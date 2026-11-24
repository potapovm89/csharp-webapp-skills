---
name: s3-storage
description: >-
  Use this skill when interacting with S3-compatible object storage (Garage, RustFS, Ceph RGW, AWS S3 and similar), including uploading, downloading, and generating presigned URLs for files.
  Triggers (RU): S3, файловое хранилище, загрузка файлов, бакет, RustFS, Garage.
---

# S3 File Storage

This skill covers working with S3-compatible object storage across all environments in the solution.

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

## Best Practices
- **Deterministic Keys**: Form object keys from entity identifiers (e.g. `$"{storeId}_{orderId}.pdf"`) so assets can be retrieved without a separate file index database.
- **Presigned URLs**: Never expose raw bucket URLs or public bucket access. Only return temporary presigned links using `GetFileLink`.
- **Stream Lifecycle**: Set `autoCloseStream: true` when passing streams that should be disposed after upload.
- **Secret Protection**: AccessKey and SecretKey must only contain dummy values in repository files (`secretName`, `secretPassword`). Real keys are injected via environment variables at runtime.
