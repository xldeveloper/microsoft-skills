---
name: azure-storage-blob-rust
description: |
  Azure Blob Storage SDK for Rust (v0.11.0). Use for uploading, downloading, and managing blobs and containers.
  Triggers: "blob storage rust", "BlobClient rust", "upload blob rust", "download blob rust", "storage container rust".
---

# Azure Blob Storage SDK for Rust

> `azure_storage_blob` v0.11.0 — Client library for Azure Blob Storage.

> **IMPORTANT:** Only use the official `azure_storage_blob` crate installed via `cargo add` from [crates.io](https://crates.io/crates/azure_storage_blob). Do NOT use the unofficial `azure_storage` or `azure_sdk_for_rust` community crates.

> **Warning:** This crate is under active development and not suitable for production environments.

## Installation

```sh
cargo add azure_storage_blob azure_identity tokio
```

## Environment Variables

```bash
AZURE_STORAGE_ENDPOINT=https://<account>.blob.core.windows.net/
```

## Authentication

```rust
use azure_storage_blob::{BlobClient, BlobClientOptions};
use azure_identity::DeveloperToolsCredential;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let credential = DeveloperToolsCredential::new(None)?;
    let blob_client = BlobClient::new(
        "https://<storage_account_name>.blob.core.windows.net/",
        "container_name",
        "blob_name",
        Some(credential),
        Some(BlobClientOptions::default()),
    )?;
    Ok(())
}
```

## Client Types

| Client                | Purpose                                   |
| --------------------- | ----------------------------------------- |
| `BlobServiceClient`   | Account-level operations, list containers |
| `BlobContainerClient` | Container operations, list blobs          |
| `BlobClient`          | Individual blob operations                |

## Upload Blob

```rust
use azure_core::http::RequestContent;
use azure_storage_blob::{BlobClient, BlobClientOptions};
use azure_identity::DeveloperToolsCredential;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let credential = DeveloperToolsCredential::new(None)?;
    let blob_client = BlobClient::new(
        "https://<storage_account_name>.blob.core.windows.net/",
        "container_name",
        "blob_name",
        Some(credential),
        Some(BlobClientOptions::default()),
    )?;

    let data = b"hello world";
    blob_client
        .upload(
            RequestContent::from(data.to_vec()),
            None,
        )
        .await?;
    Ok(())
}
```

## Download Blob / Get Properties

```rust
// Get blob properties
let blob_properties = blob_client.get_properties(None).await?;

// Download blob content
let response = blob_client.download(None).await?;
let content = response.into_body().collect_bytes().await?;
```

## Delete Blob

```rust
blob_client.delete(None).await?;
```

## Container Operations

```rust
use azure_storage_blob::BlobContainerClient;

let container_client = BlobContainerClient::new(
    "https://<account>.blob.core.windows.net/",
    "container-name",
    Some(credential),
    None,
)?;

// Create container
container_client.create(None).await?;

// List blobs
let mut pager = container_client.list_blobs(None)?;
while let Some(blob) = pager.try_next().await? {
    println!("Blob: {}", blob.name);
}
```

## RBAC Permissions

For Entra ID auth, assign one of these roles:

| Role                            | Access                     |
| ------------------------------- | -------------------------- |
| `Storage Blob Data Reader`      | Read-only                  |
| `Storage Blob Data Contributor` | Read/write                 |
| `Storage Blob Data Owner`       | Full access including RBAC |

## Best Practices

1. **Use Entra ID auth** — `DeveloperToolsCredential` for dev, `ManagedIdentityCredential` for production
2. **Use `RequestContent::from()`** — to wrap upload data
3. **Assign RBAC roles** — ensure "Storage Blob Data Contributor" for write access
4. **Reuse clients** — clients are thread-safe

## Reference Links

| Resource      | Link                                           |
| ------------- | ---------------------------------------------- |
| API Reference | https://docs.rs/azure_storage_blob             |
| crates.io     | https://crates.io/crates/azure_storage_blob    |
