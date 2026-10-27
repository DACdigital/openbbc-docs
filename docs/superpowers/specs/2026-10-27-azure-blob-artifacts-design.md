# azure-blob-artifacts — `azure_blob` artifact-store kind in open-bbcd

**Date:** 2026-10-27
**Status:** Draft — pending `/spec-review`
**Modularity attachment:** `docs/architecture/current/modularity/open-bbcd/artifacts/` (L2).

**Arch-of-record:**
- "Artifact-store kinds are versioned via the `kind` env-var value … adding a new kind is a
  code change" in `constraints.md`.
- Object store row in `c4/integrations.md`.
- Glossary term "Artifact-store adapter".
- `ddd/contexts/artifacts.md`, `ddd/context-map.md` (adapter as ACL).
- Introduced by `docs/superpowers/specs/2026-09-28-chat-artifacts-design.md`.

**Grounded against:** OpenBBC `main` @ `b2cca70`;
`github.com/Azure/azure-sdk-for-go/sdk/storage/azblob` `v1.8.2`. File references are
relative to `open-bbcd/` unless they start with `deploy/` or `docker-compose.yml`, which are
relative to the OpenBBC repo root.

## Business value / Why

Chat and deployed-runtime artifacts (uploaded files, tool-produced images) are stored as
bytes in a deployer-configured object store. Today the only adapter is `s3_compatible`
(`internal/artifacts/s3.go`). Azure Blob Storage has no S3 API. A deployer on Azure (the
AI-sales environment uses storage account `staisalesdev`) would have to run and secure an
S3 gateway (s3proxy, or MinIO's deprecated gateway mode) next to the daemon.

This spec adds a native `azure_blob` kind. An Azure deployer then sets env vars and gets
artifacts working, with no extra service to run.

Success is judged by:
- A deployer sets `ARTIFACT_STORE_<ID>_KIND=azure_blob` plus account name, key and container.
  The daemon then probes the container at boot. Upload, retrieval (302 to a SAS URL) and
  LLM multimodal rendering behave exactly as they do on `s3_compatible`.
- Existing `s3_compatible` deployments are unaffected.

## Change level

**C2.** It extends the deployer-facing env-var contract with a new `KIND` value and its own
option keys. It adds an external integration (Azure Blob REST API) and a new third-party
dependency (Azure SDK for Go). It also changes stated facts in
`docs/architecture/current/`, which name `s3_compatible` as the only kind.

## Scope

### In scope

- New adapter `internal/artifacts/azure.go` implementing `ArtifactStore` for kind
  `azure_blob`, registered in `adapterFactories` and listed in `supportedKindsList()`
  (`internal/artifacts/registry.go`).
- Shared Key auth (storage account name + account key).
- `PreferredDelivery() = DeliverySignedURL`: reads are served as a 302 to a read-only
  Blob service SAS URL signed with the account key, carrying `SignOptions` as SAS response
  overrides (`rsct` = Content-Type, `rscd` = Content-Disposition).
- Boot `Probe` with the same contract as `s3_compatible`.
- An opt-in integration test against Azurite (local emulator) or a real account, plus a
  manual live smoke test against `staisalesdev/session-artifacts`.
- An `azurite` service in `docker-compose.yml` under the `artifacts` profile, started with
  `--skipApiVersionCheck` so the SDK's newer `x-ms-version` is accepted. Also example env
  blocks in the `docker-compose.yml` comments and `open-bbcd/.env.example`, using the Azurite
  dev account `devstoreaccount1` or placeholders and never a real account name. A comment
  notes that `http://azurite:10000` SAS redirects resolve only inside the compose network,
  the same caveat as `http://minio:9000`.

### Out of scope

- Entra ID / managed identity / workload identity auth, and user-delegation SAS.
- Connection-string or SAS-token options. Name + key covers every Shared Key deployment,
  and a connection string holds no information beyond them.
- Creating the container at boot. The deployer provisions it, as with S3 buckets.
- Migrating blobs between stores; `DeliveryBytes` mode for Azure.
- Any change to the retrieval route, upload route, render path, Postgres schema or
  `artifact_ref` block shape.

## Contracts

### Env vars (parsed generically by `internal/config/config.go`; validated by the adapter)

| Var | Required | Meaning |
|---|---|---|
| `ARTIFACT_STORE_<ID>_KIND=azure_blob` | yes | Selects this adapter. |
| `ARTIFACT_STORE_<ID>_ACCOUNT_NAME` | yes | Storage account name (e.g. `staisalesdev`). |
| `ARTIFACT_STORE_<ID>_ACCOUNT_KEY` | yes | Base64 account key. Secret, in the same class as `_SECRET_KEY`. Construction fails if it does not base64-decode. |
| `ARTIFACT_STORE_<ID>_CONTAINER` | yes | Blob container name. Must already exist. |
| `ARTIFACT_STORE_<ID>_ENDPOINT` | no | Blob service URL. Default `https://<ACCOUNT_NAME>.blob.core.windows.net/`. Set it for Azurite (`http://127.0.0.1:10000/<account>/`) or sovereign clouds. Must be `https://`, or `http://` only to a loopback host or a single-label host (docker-compose service name). |

Error messages are prefixed `azure_blob:` and name the missing or invalid key, mirroring
`s3_compatible:`. All other `ARTIFACT_*` vars (`DEFAULT`, `MAX_UPLOAD_MB`,
`SIGNED_URL_TTL_SECONDS`, `MAX_PENDING`) are unchanged and apply to this kind as-is.
`config.go` needs no change. It already groups arbitrary `<FIELD>` keys into
`Options` per store.

### Go internal contract (`ArtifactStore` method mapping)

| Method | Azure behaviour |
|---|---|
| `Kind()` | `"azure_blob"` |
| `PreferredDelivery()` | `DeliverySignedURL` |
| `Put(uri, mime, r, size)` | Block-blob upload (`UploadStream`) of the blob named `uri` verbatim (`sha256/<hex>`), with `BlobContentType=mime`. Returns `{URI, SizeBytes: size, Sha256: hex from uri}`. |
| `Get(uri)` | `DownloadStream` body. Any 404 (`BlobNotFound` or `ContainerNotFound`) → wrapped `ErrBlobMissing`. |
| `Sign(uri, ttl, opts)` | Local, no network: Blob SAS, permission `r`, protocol HTTPS (HTTPS+HTTP only when ENDPOINT is http), start = now − 5 min (clock skew), expiry = now + ttl (`ttl<=0` → constructor TTL). `opts.ContentType` → `rsct`, `opts.ContentDisposition` → `rscd`. Like S3 presign, it does not check that the blob exists. |
| `Stat(uri)` | `GetProperties`. Any 404 (`BlobNotFound` or `ContainerNotFound`) → `{Exists:false}, nil`. This is deliberately lenient, matching `isS3NotFound` treating `NoSuchBucket` as not-found (`internal/artifacts/s3.go`). A deleted container surfaces as 410 on retrieval. `Probe` catches it at boot. Success → `Exists, MIME=ContentType, SizeBytes=ContentLength, Sha256=hex from uri`. |
| `Delete(uri)` | `Delete` with snapshots included. `BlobNotFound` → nil, which keeps it idempotent like S3 `RemoveObject`. |
| `Probe()` | Put `_probe/<rand>` → Stat → Delete. Then Stat of `_probe/missing-<rand>` must report `Exists:false`. Any failure yields `ErrProbeFailed` naming the account, the container and the Azure error code. For the common codes it adds a hint: `AuthenticationFailed` → wrong or rotated key; `AuthorizationFailure` → storage-account firewall/network rules block the daemon; `KeyBasedAuthenticationNotPermitted` → Shared Key is disabled on the account; `ContainerNotFound` → the container must exist. |

`Stat`'s `Sha256` derives from the content-addressed URI and not from the store's ETag.
That is accurate for this kind, because blobs are only written under `sha256/<hex>`.

### Unchanged contracts

REST routes and their status codes (302 / 410 / 502), the `artifact_ref` block,
`chat_session_artifacts` / `deployed_session_artifacts`, and the `llm.ArtifactFetcher` shim
are all unchanged.

### Deviations from the arch-of-record (for `/arch-review` to sync)

`current/` names `s3_compatible` as the only shipped kind, and lists only S3-shaped config
(endpoint, bucket, region, access/secret key, path-style). These places need
`azure_blob` and its keys (account name/key, container, endpoint) added:
- `c4/integrations.md` (Object store row and the "Kinds are versioned" bullet)
- `constraints.md:110`, `constraints.md:127-128`
- `assumptions.md:90`
- `glossary.md` ("Artifact-store adapter")
- `bizbok/capabilities.md:166-169`
- `bizbok/information-map.md` (Artifact store row: `kind ∈ {s3_compatible, azure_blob}`)
- `ddd/contexts/artifacts.md:21`, `ddd/context-map.md` (artifacts ↔ object store row)
- `c4/context.md:62`
- `modularity/open-bbcd/artifacts/README.md:45`

The probe wording "for s3_compatible this needs s3:ListBucket" gains an Azure equivalent:
Shared Key always grants this, so there is no extra permission to add.
`c4/integrations.md:11` also needs its Protocol cell ("S3 API over HTTPS") generalised to
include the Azure Blob REST API, and its Auth cell, which lists only S3 env vars, extended
with the `azure_blob` keys. `store.go`'s S3-specific `Probe` doc comment is generalised in
code.

## Acceptance criteria

### Construction / config (unit tests)

- [ ] Missing `ACCOUNT_NAME`, `ACCOUNT_KEY` or `CONTAINER` each fails `NewAzureBlob` with an
      `azure_blob:` error naming that key.
- [ ] A non-base64 `ACCOUNT_KEY` fails construction with an error naming `ACCOUNT_KEY`.
      The key value never appears in the error.
- [ ] With `ENDPOINT` unset, the service URL is `https://<ACCOUNT_NAME>.blob.core.windows.net/`.
- [ ] `ENDPOINT=http://example.com/` fails construction. `ENDPOINT=http://127.0.0.1:10000/devstoreaccount1/`
      and `ENDPOINT=http://azurite:10000/devstoreaccount1/` are accepted. `ENDPOINT=ftp://x` fails.
- [ ] `signTTL <= 0` fails construction.
- [ ] `artifacts.Load` with `KIND=azure_blob` builds a registry whose store reports
      `Kind()=="azure_blob"` and `PreferredDelivery()==DeliverySignedURL`. An unknown-kind error
      lists `s3_compatible, azure_blob`.
- [ ] `Sign` (offline, no network) returns a URL on the configured endpoint, under
      `<container>/sha256/<hex>`. Its query has `sp=r`, `se` about ttl from now, `st` about 5 min in the
      past, and `rsct`/`rscd` equal to the given options, or absent when empty.

### Integration (`TestAzureBlob_Integration`, opt-in via `AZURE_BLOB_*` env; skipped otherwise)

- [ ] Passes against Azurite (`docker compose --profile artifacts up -d azurite`). The test itself
      creates the container if absent, so no `az` CLI bootstrap is needed. This only happens in
      test code: the adapter never creates containers.
- [ ] Passes against the real `staisalesdev` account, container `session-artifacts`.
- [ ] Sequence: `Probe` succeeds. `Stat` of a fresh URI reports not-exists. `Put` and then `Stat` reports
      exists with matching MIME and size. `Get` returns identical bytes. A plain `http.Get` of
      `Sign(..., {ContentType:"text/plain", ContentDisposition:"attachment; filename=\"a.txt\""})`
      returns 200, identical bytes and those two response headers. `Delete` then `Stat` reports
      not-exists. `Get` of a deleted URI returns `errors.Is(err, ErrBlobMissing)`. A second `Delete`
      returns nil.
- [ ] `Probe` with a wrong (valid base64) key fails with `ErrProbeFailed`. `Probe` against a
      non-existent container fails with `ErrProbeFailed` mentioning the container.

### Live smoke (manual)

- [ ] `open-bbcd serve` with `ARTIFACT_STORE_AZ_*` pointing at `staisalesdev/session-artifacts` and
      `ARTIFACT_STORE_DEFAULT=AZ` boots and logs a successful probe.
- [ ] A file uploaded in a BO chat session appears as blob `sha256/<hex>` in the container. Its
      retrieval route answers 302 to a `*.blob.core.windows.net` SAS URL that serves the bytes
      with the expected Content-Type.
- [ ] An image artifact is rendered to the model in the next turn (multimodal path), with no
      error logged.
- [ ] No credential is committed. Smoke credentials live only in a gitignored env file.
- [ ] After the smoke test, the `staisalesdev` account key used during design is rotated. The new
      key never leaves the gitignored env file or the deployer's secret store.

### Docs

- [ ] `.env.example` and `docker-compose.yml` comments show an `azure_blob` example block.
- [ ] `docs/architecture/current/` synced via `/arch-review` (see Deviations).

## Risks & assumptions

### Assumptions

- Azure answers `GetProperties` on a missing blob with 404 `BlobNotFound` when using Shared Key auth. Shared
  Key is account-wide, so there is no S3-style "403 without ListBucket" case. `Probe` still verifies it.
- Blob SAS `rsct`/`rscd` overrides are honoured for service SAS signed with the account key
  (documented behaviour). The integration test verifies it.
- Azurite accepts the same SAS and Shared Key signing as the real service, using its well-known dev
  account `devstoreaccount1`.
- The azblob module (`azcore` + `azblob`) builds with `CGO_ENABLED=0` on the repo's toolchain
  (Go 1.26 today, 1.27+ once bifrost lands). It is pure Go.

### Risks

- **Account key blast radius.** The key grants full control of the whole storage account, not just the
  container. Mitigation: document that deployers should use a dedicated storage account for
  artifacts. Entra ID / user-delegation SAS (out of scope) is the long-term fix.
- **Shared Key disabled on the account.** Azure Policy and newer account defaults increasingly set
  `allowSharedKeyAccess=false`, and then this adapter cannot work. `Probe` fails boot with
  `KeyBasedAuthenticationNotPermitted`. Mitigation: a dedicated artifacts account with Shared
  Key enabled. Entra ID support is the long-term fix.
- **Clock skew.** A SAS is rejected if `st` is in the future on Azure's clock. Mitigated by backdating
  `st` by 5 minutes.
- **SAS URLs are bearer tokens.** Exposure is the same as S3 presigned URLs. They are bounded by
  `ARTIFACT_SIGNED_URL_TTL_SECONDS` and read-only on one blob.
- **Dependency weight.** The Azure SDK adds `azcore` and its transitive dependencies to the binary. This is
  accepted, in the same way `minio-go` was accepted for S3.
- **Credentials pasted during design.** The `staisalesdev` key was shared in a working session.
  It is rotated unconditionally after the smoke test (see Live smoke AC).

## Cross-references

- `docs/superpowers/specs/2026-09-28-chat-artifacts-design.md` — adapter interface and registry.
- `docs/superpowers/specs/2026-09-30-deployed-runtime-artifacts-design.md` — deployed-session artifacts.
- `docs/architecture/current/modularity/open-bbcd/artifacts/README.md`
