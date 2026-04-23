---
name: azure-cosmos-rust
description: |
  Azure Cosmos DB SDK for Rust (v0.32.0, NoSQL API). Use for document CRUD, containers, and globally distributed data.
  Triggers: "cosmos db rust", "CosmosClient rust", "document crud rust", "NoSQL rust", "partition key".
---

# Azure Cosmos DB SDK for Rust

> `azure_data_cosmos` v0.32.0 — Client library for Azure Cosmos DB NoSQL API.

> **IMPORTANT:** Only use the official `azure_data_cosmos` crate installed via `cargo add` from [crates.io](https://crates.io/crates/azure_data_cosmos). Do NOT use the unofficial `azure_cosmos` or `azure_sdk_for_rust` community crates.

## Installation

```sh
cargo add azure_data_cosmos azure_identity tokio
```

## Environment Variables

```bash
COSMOS_ENDPOINT=https://<account>.documents.azure.com/
```

## Authentication

```rust
use azure_identity::DeveloperToolsCredential;
use azure_data_cosmos::{
    CosmosClient, CosmosAccountReference, CosmosAccountEndpoint, RoutingStrategy,
};

async fn create_client() -> Result<CosmosClient, Box<dyn std::error::Error>> {
    let credential: std::sync::Arc<dyn azure_core::credentials::TokenCredential> =
        DeveloperToolsCredential::new(None)?;
    let endpoint: CosmosAccountEndpoint = "https://myaccount.documents.azure.com/"
        .parse()?;
    let account = CosmosAccountReference::with_credential(endpoint, credential);
    let cosmos_client = CosmosClient::builder()
        .build(account, RoutingStrategy::ProximityTo("East US".into()))
        .await?;
    Ok(cosmos_client)
}
```

## Client Hierarchy

| Client            | Purpose                   | Access                                  |
| ----------------- | ------------------------- | --------------------------------------- |
| `CosmosClient`    | Account-level operations  | `CosmosClient::builder().build()`       |
| `DatabaseClient`  | Database operations       | `client.database_client("db")`          |
| `ContainerClient` | Container/item operations | `database.container_client("c").await?` |

## CRUD Operations

```rust
use serde::{Serialize, Deserialize};
use azure_data_cosmos::CosmosClient;

#[derive(Serialize, Deserialize)]
struct Item {
    pub id: String,
    pub partition_key: String,
    pub value: String,
}

async fn crud_example(cosmos_client: CosmosClient) -> Result<(), Box<dyn std::error::Error>> {
    let item = Item {
        id: "1".into(),
        partition_key: "partition1".into(),
        value: "2".into(),
    };

    let container = cosmos_client
        .database_client("myDatabase")
        .container_client("myContainer")
        .await?;

    // Create an item
    container.create_item("partition1", item, None).await?;

    // Read an item
    let item_response = container.read_item("partition1", "1", None).await?;
    let mut item: Item = item_response.into_model()?;

    item.value = "3".into();

    // Replace an item
    container.replace_item("partition1", "1", item, None).await?;

    // Delete an item
    container.delete_item("partition1", "1", None).await?;
    Ok(())
}
```

## Key Auth (Optional)

Enable account key authentication with the feature flag:

```sh
cargo add azure_data_cosmos --features key_auth
```

## Best Practices

1. **Use Entra ID auth** — `DeveloperToolsCredential` for dev, `ManagedIdentityCredential` for production
2. **Reuse `CosmosClient`** — clients are thread-safe; create once, share across threads
3. **Use `RoutingStrategy::ProximityTo`** — route to the nearest region for lowest latency
4. **Partition keys** — always specify partition key for item operations

## Reference Links

| Resource      | Link                                                                      |
| ------------- | ------------------------------------------------------------------------- |
| API Reference | https://docs.rs/azure_data_cosmos                                         |
| crates.io     | https://crates.io/crates/azure_data_cosmos                                |
