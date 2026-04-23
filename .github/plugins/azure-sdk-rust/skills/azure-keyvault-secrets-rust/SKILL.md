---
name: azure-keyvault-secrets-rust
description: |
  Azure Key Vault Secrets SDK for Rust (v0.13.0). Use for storing and retrieving secrets, passwords, and API keys.
  Triggers: "keyvault secrets rust", "SecretClient rust", "get secret rust", "set secret rust".
---

# Azure Key Vault Secrets SDK for Rust

> `azure_security_keyvault_secrets` v0.13.0 — Secure storage for passwords, API keys, and connection strings.

> **IMPORTANT:** Only use the official `azure_security_keyvault_secrets` crate installed via `cargo add` from [crates.io](https://crates.io/crates/azure_security_keyvault_secrets). Do NOT use unofficial or community crates.

## Installation

```sh
cargo add azure_security_keyvault_secrets azure_identity tokio futures
```

## Environment Variables

```bash
AZURE_KEYVAULT_URL=https://<vault-name>.vault.azure.net/
```

## Client Setup

```rust
use azure_identity::DeveloperToolsCredential;
use azure_security_keyvault_secrets::SecretClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let credential = DeveloperToolsCredential::new(None)?;
    let client = SecretClient::new(
        "https://<your-key-vault-name>.vault.azure.net/",
        credential.clone(),
        None,
    )?;

    let secret = client
        .get_secret("secret-name", None)
        .await?
        .into_model()?;
    println!("Secret: {:?}", secret.value);

    Ok(())
}
```

## Set Secret

```rust
use azure_security_keyvault_secrets::{models::SetSecretParameters, ResourceExt};

let secret_set_parameters = SetSecretParameters {
    value: Some("secret-value".into()),
    ..Default::default()
};

let secret = client
    .set_secret("secret-name", secret_set_parameters.try_into()?, None)
    .await?
    .into_model()?;

println!(
    "Secret Name: {:?}, Value: {:?}, Version: {:?}",
    secret.resource_id()?.name,
    secret.value,
    secret.resource_id()?.version
);
```

## Update Secret Properties

```rust
use azure_security_keyvault_secrets::models::UpdateSecretPropertiesParameters;
use std::collections::HashMap;

let secret_update_parameters = UpdateSecretPropertiesParameters {
    content_type: Some("text/plain".into()),
    tags: Some(HashMap::from_iter(vec![(
        "tag-name".into(),
        "tag-value".into(),
    )])),
    ..Default::default()
};

client
    .update_secret_properties(
        "secret-name",
        secret_update_parameters.try_into()?,
        None,
    )
    .await?
    .into_model()?;
```

## Delete Secret

```rust
client.delete_secret("secret-name", None).await?;
```

## List Secrets

Note: `list_secret_properties` returns a pager directly (no `.into_stream()`).

```rust
use azure_security_keyvault_secrets::ResourceExt;
use futures::TryStreamExt;

let mut pager = client.list_secret_properties(None)?;
while let Some(secret) = pager.try_next().await? {
    let name = secret.resource_id()?.name;
    println!("Found Secret with Name: {}", name);
}
```

## Error Handling

```rust
use azure_core::{error::ErrorKind, http::StatusCode};

match client.get_secret("secret-name", None).await {
    Ok(response) => println!("Secret: {:?}", response.into_model()?.value),
    Err(e) => match e.kind() {
        ErrorKind::HttpResponse { status, error_code, .. }
            if *status == StatusCode::NotFound =>
        {
            println!("Secret not found");
            if let Some(code) = error_code {
                println!("ErrorCode: {}", code);
            }
        }
        _ => println!("Error: {e:?}"),
    },
}
```

## Best Practices

1. **Use Entra ID auth** — `DeveloperToolsCredential` for dev, `ManagedIdentityCredential` for production
2. **Use `..Default::default()`** — for struct update syntax on all model types
3. **Use `ResourceExt`** — to extract resource name/version from secret IDs
4. **Reuse clients** — `SecretClient` is thread-safe

## Reference Links

| Resource      | Link                                                          |
| ------------- | ------------------------------------------------------------- |
| API Reference | https://docs.rs/azure_security_keyvault_secrets               |
| crates.io     | https://crates.io/crates/azure_security_keyvault_secrets      |
