---
name: azure-keyvault-keys-rust
description: |
  Azure Key Vault Keys SDK for Rust (v0.13.0). Use for creating, managing, and using cryptographic keys including RSA, EC, and HSM-protected keys.
  Triggers: "keyvault keys rust", "KeyClient rust", "create key rust", "encrypt rust", "wrap key rust", "sign rust".
---

# Azure Key Vault Keys SDK for Rust

> `azure_security_keyvault_keys` v0.13.0 — Secure storage and management of cryptographic keys.

> **IMPORTANT:** Only use the official `azure_security_keyvault_keys` crate installed via `cargo add` from [crates.io](https://crates.io/crates/azure_security_keyvault_keys). Do NOT use unofficial or community crates.

## Installation

```sh
cargo add azure_security_keyvault_keys azure_identity tokio futures
```

## Environment Variables

```bash
AZURE_KEYVAULT_URL=https://<vault-name>.vault.azure.net/
```

## Client Setup

```rust
use azure_identity::DeveloperToolsCredential;
use azure_security_keyvault_keys::KeyClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let credential = DeveloperToolsCredential::new(None)?;
    let client = KeyClient::new(
        "https://<your-key-vault-name>.vault.azure.net/",
        credential.clone(),
        None,
    )?;

    let key = client
        .get_key("key-name", None)
        .await?
        .into_model()?;
    println!("JWT: {:?}", key.key);

    Ok(())
}
```

## Create Key

```rust
use azure_security_keyvault_keys::{
    models::{CreateKeyParameters, CurveName, KeyType},
    ResourceExt,
};

// Create an EC key
let body = CreateKeyParameters {
    kty: Some(KeyType::Ec),
    curve: Some(CurveName::P256),
    ..Default::default()
};

let key = client
    .create_key("key-name", body.try_into()?, None)
    .await?
    .into_model()?;

println!(
    "Key Name: {:?}, Type: {:?}, Version: {:?}",
    key.resource_id()?.name,
    key.key.as_ref().map(|k| k.kty.as_ref()),
    key.resource_id()?.version,
);
```

## Update Key Properties

```rust
use azure_security_keyvault_keys::models::UpdateKeyPropertiesParameters;
use std::collections::HashMap;

let key_update_parameters = UpdateKeyPropertiesParameters {
    tags: Some(HashMap::from_iter(vec![("tag-name".into(), "tag-value".into())])),
    ..Default::default()
};

client
    .update_key_properties("key-name", key_update_parameters.try_into()?, None)
    .await?
    .into_model()?;
```

## Delete Key

```rust
client.delete_key("key-name", None).await?;
```

## List Keys

```rust
use azure_security_keyvault_keys::ResourceExt;
use futures::TryStreamExt;

let mut pager = client.list_key_properties(None)?.into_stream();
while let Some(key) = pager.try_next().await? {
    let name = key.resource_id()?.name;
    println!("Found Key with Name: {}", name);
}
```

## Encrypt and Decrypt (Wrap/Unwrap)

Key Vault performs crypto operations server-side — the private key never leaves the HSM:

```rust
use azure_security_keyvault_keys::{
    models::{
        CreateKeyParameters, EncryptionAlgorithm, KeyOperationParameters, KeyType,
    },
    ResourceExt,
};
use rand::random;

// Create a key encryption key (KEK) using RSA
let body = CreateKeyParameters {
    kty: Some(KeyType::Rsa),
    key_size: Some(2048),
    ..Default::default()
};

let key = client
    .create_key("key-name", body.try_into()?, None)
    .await?
    .into_model()?;
let key_version = key.resource_id()?.version.expect("key version required");

// Generate a symmetric data encryption key (DEK)
let dek = random::<u32>().to_le_bytes().to_vec();

// Wrap the DEK
let mut parameters = KeyOperationParameters {
    algorithm: Some(EncryptionAlgorithm::RsaOaep256),
    value: Some(dek.clone()),
    ..Default::default()
};
let wrapped = client
    .wrap_key("key-name", &key_version, parameters.clone().try_into()?, None)
    .await?
    .into_model()?;

assert!(matches!(wrapped.result.as_ref(), Some(result) if !result.is_empty()));

// Unwrap the DEK
parameters.value = wrapped.result;
let unwrapped = client
    .unwrap_key("key-name", &key_version, parameters.try_into()?, None)
    .await?
    .into_model()?;

assert_eq!(unwrapped.result.as_ref(), Some(&dek));
```

## Key Types

| Type   | Use Case                      | Parameter           |
| ------ | ----------------------------- | ------------------- |
| EC     | Signing, key agreement        | `KeyType::Ec`       |
| RSA    | Encryption, signing, wrapping | `KeyType::Rsa`      |
| Oct    | Symmetric operations (HSM)    | `KeyType::Oct`      |
| EC-HSM | HSM-protected EC keys         | `KeyType::EcHsm`   |
| RSA-HSM| HSM-protected RSA keys        | `KeyType::RsaHsm`  |

## Best Practices

1. **Use Entra ID auth** — `DeveloperToolsCredential` for dev, `ManagedIdentityCredential` for production
2. **Use `..Default::default()`** — for struct update syntax on all model types
3. **Use `ResourceExt`** — to extract key name/version from key IDs
4. **Reuse clients** — `KeyClient` is thread-safe

## Reference Links

| Resource      | Link                                                      |
| ------------- | --------------------------------------------------------- |
| API Reference | https://docs.rs/azure_security_keyvault_keys              |
| crates.io     | https://crates.io/crates/azure_security_keyvault_keys     |
