---
name: s3-storage
description: >-
  Use this skill when interacting with S3-compatible object storage (Garage, RustFS, Ceph RGW, AWS S3 and similar), including uploading, downloading, and generating presigned URLs for files.
  Triggers (RU): S3, файловое хранилище, загрузка файлов, бакет, RustFS, Garage.
---

# S3 File Storage

This skill covers working with S3-compatible object storage across all environments in the solution.

## Choose the Operation

1. Find the existing client registration and `FileStorageOptions`. Confirm the bucket, region, service endpoint, presigned-link endpoint and the caller's permissions. Use an isolated bucket and fixture credentials for verification.
2. Use `ISingleBucketFileClient` for the configured bucket and `IFileClient` for operations on several buckets. Derive keys from trusted identifiers and pass the caller's cancellation token to storage calls.
3. Define the outcomes for the changed operation before wiring it into a Story or API. An empty listing returns an empty collection. For a read, map a confirmed missing key to the application's not-found outcome. Keep access denied, cancellation and storage faults distinct. For a presigned link, verify the caller can access the object before signing and use the endpoint reachable by the recipient.

**Scope example** — An attachment downloader fetches files over HTTP and enforces a cumulative byte limit and URL policy. Use HTTP client and security rules for that behavior. Apply the S3 skill when adding application-owned object storage.

## Environment Setup
- **Deployed environments**: any S3-compatible storage (e.g. Garage, Ceph RGW, AWS S3). Which one is used is defined by the solution's infrastructure, not by the code: the code works only through the S3 API.
- **Local Aspire / Integration Tests**: RustFS (`rustfs/rustfs`, Apache-2.0) with a pinned image tag. Do not use MinIO for new setups: its images are no longer published openly.
- **SDK**: Standard `AWSSDK.S3` via abstraction interfaces in `DataAccess/S3`.
- **AWSSDK.S3 4.x**: collection properties of responses (`ListBucketsResponse.Buckets`, `ListObjectsV2Response.S3Objects`, `CommonPrefixes`, `DeleteObjectsResponse.DeletedObjects`, etc.) are `null` when empty, not an empty list (v3 returned empty lists). Always read them with `?.` / `?? []`; an unguarded `.Select` crashes on an empty bucket or an empty storage.

## Client Interfaces in `DataAccess/S3`
- `ISingleBucketFileClient`: Primary high-level client operating on the default bucket (`MainBucket` from `FileStorageOptions`):
  - `PutAsync(key, stream, autoCloseStream, ct)`: Uploads a stream.
  - `GetAsync(key, ct)` / `GetAsync(key, versionId, ct)`: Downloads a stream.
  - `DeleteAsync(key, ct)`: Deletes an object.
  - `GetFileLink(fileName)`: Generates a temporary presigned URL (`LinkLiveTime`).
- `IFileClient`: Low-level multi-bucket client when multiple distinct buckets are involved.

## Configuration & DI Registration

```csharp
// Program.cs
builder.Services.AddFileClient(builder.Configuration.GetRequiredSection(nameof(FileStorageOptions)).Bind);
builder.Services.AddScoped<ISingleBucketFileClient, SingleBucketFileClient>();
```

Configuration section in JSON:
```json
"FileStorageOptions": {
  "Endpoint": "http://localhost:9010",
  "AccessKey": "secretName",
  "SecretKey": "secretPassword",
  "Region": "garage",
  "MainBucket": "testbucket",
  "LinkLiveTime": "01:00:00"
}
```

- `Region` must match the region configured on the storage server (e.g. `s3_region` in Garage, default `garage`; RustFS accepts any region). A mismatch breaks request signing and presigned URLs. The value above is an example for a Garage-based environment.
- The `AmazonS3Config` behind `AddFileClient` must use `ForcePathStyle = true`: self-hosted S3 implementations (Garage, RustFS, Ceph) do not support virtual-hosted-style bucket addressing by default. When changing the client registration, keep this flag.
- Presigned URLs are signed for `Endpoint`. If clients reach the storage through a different public host, the link must be generated for that public host, otherwise the signature is invalid.
- Use the provider's error contract when distinguishing missing objects from permission failures. For Amazon S3, a missing object may return 403 when the caller lacks `s3:ListBucket`. Treat 403 as access denied unless the provider supplies reliable evidence of absence. [Amazon S3 GetObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html)

## Best Practices
- **Deterministic Keys**: Form object keys from entity identifiers (e.g. `$"{storeId}_{orderId}.pdf"`) so assets can be retrieved without a separate file index database.
- **Presigned URLs**: Never expose raw bucket URLs or public bucket access. Only return temporary presigned links using `GetFileLink`.
- **Stream Lifecycle**: Set `autoCloseStream: true` when passing streams that should be disposed after upload.
- **Secret Protection**: AccessKey and SecretKey must only contain dummy values in repository files (`secretName`, `secretPassword`). Real keys are injected via environment variables at runtime.

## Completion Evidence

- With the registered client and an isolated private bucket, write known bytes with `PutAsync`, read them with `GetAsync`, and assert byte-for-byte equality. When adding or changing presigned access, fetch the short-lived GET link as-is from the recipient's side without SDK credentials. Assert the same bytes, denial of the equivalent unsigned request, and a signed lifetime within `LinkLiveTime`.
- Exercise a missing key with a fixture that returns a confirmed absence, and separately exercise an unavailable storage endpoint. Assert the documented not-found and availability outcomes respectively. If changing a listing, include an empty bucket and assert an empty collection. A failed upload must leave any affected application record outside its published or ready state.
- Record the test command and outcomes. Omit signed URLs and credentials from evidence.
