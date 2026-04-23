---
name: azure-keyvault-certificates-rust
description: |
  Azure Key Vault Certificates SDK for Rust (v0.12.0). Use for creating, managing, and using X.509 certificates including self-signed and CA-issued.
  Triggers: "keyvault certificates rust", "CertificateClient rust", "create certificate rust", "sign with certificate rust".
---

# Azure Key Vault Certificates SDK for Rust

> `azure_security_keyvault_certificates` v0.12.0 — Manage X.509 certificates for TLS/SSL, code signing, and authentication.

> **IMPORTANT:** Only use the official `azure_security_keyvault_certificates` crate installed via `cargo add` from [crates.io](https://crates.io/crates/azure_security_keyvault_certificates). Do NOT use unofficial or community crates.

## Installation

```sh
cargo add azure_security_keyvault_certificates azure_identity tokio futures
```

## Environment Variables

```bash
AZURE_KEYVAULT_URL=https://<vault-name>.vault.azure.net/
```

## Client Setup

```rust
use azure_identity::DeveloperToolsCredential;
use azure_security_keyvault_certificates::CertificateClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let credential = DeveloperToolsCredential::new(None)?;
    let client = CertificateClient::new(
        "https://<your-key-vault-name>.vault.azure.net/",
        credential.clone(),
        None,
    )?;

    let cert = client
        .get_certificate("cert-name", None)
        .await?
        .into_model()?;
    println!("Certificate: {:?}", cert.id);

    Ok(())
}
```

## Create Self-Signed Certificate

Creating a certificate is a long-running operation (LRO) using `Poller`:

```rust
use azure_security_keyvault_certificates::{
    models::{
        CertificateCreateParameters, IssuerParameters, SecretProperties,
        SubjectAlternativeNames, X509CertificateProperties,
    },
    ResourceExt,
};

let body = CertificateCreateParameters {
    certificate_policy: Some(Box::new(
        azure_security_keyvault_certificates::models::CertificatePolicy {
            issuer_parameters: Some(IssuerParameters {
                name: Some("Self".into()),
                ..Default::default()
            }),
            secret_properties: Some(SecretProperties {
                content_type: Some("application/x-pkcs12".into()),
                ..Default::default()
            }),
            x509_certificate_properties: Some(X509CertificateProperties {
                subject: Some("CN=example.com".into()),
                subject_alternative_names: Some(SubjectAlternativeNames {
                    dns_names: Some(vec!["example.com".into()]),
                    ..Default::default()
                }),
                ..Default::default()
            }),
            ..Default::default()
        },
    )),
    ..Default::default()
};

// Start the LRO
let poller = client
    .create_certificate("cert-name", body.try_into()?, None)
    .await?;

// Wait for completion
let cert = poller.wait().await?.into_model()?;
println!(
    "Certificate Name: {:?}, Version: {:?}",
    cert.resource_id()?.name,
    cert.resource_id()?.version,
);
```

## Update Certificate Properties

```rust
use azure_security_keyvault_certificates::models::CertificateUpdateParameters;
use std::collections::HashMap;

let update_params = CertificateUpdateParameters {
    tags: Some(HashMap::from_iter(vec![("env".into(), "prod".into())])),
    ..Default::default()
};

client
    .update_certificate("cert-name", update_params.try_into()?, None)
    .await?
    .into_model()?;
```

## Delete Certificate

```rust
client.delete_certificate("cert-name", None).await?;
```

## List Certificates

```rust
use azure_security_keyvault_certificates::ResourceExt;
use futures::TryStreamExt;

let mut pager = client.list_certificate_properties(None)?.into_stream();
while let Some(cert) = pager.try_next().await? {
    let name = cert.resource_id()?.name;
    println!("Found Certificate: {}", name);
}
```

## Key Operations Using Certificates

Certificates in Key Vault have an associated key. Use the Key Vault Keys SDK to perform crypto operations:

```rust
use azure_security_keyvault_keys::{
    models::{KeySignParameters, SignatureAlgorithm},
    KeyClient,
};

let key_client = KeyClient::new(
    "https://<your-key-vault-name>.vault.azure.net/",
    credential.clone(),
    None,
)?;

// Sign with the certificate's EC key
let digest = vec![0u8; 32]; // SHA-256 digest
let sign_params = KeySignParameters {
    algorithm: Some(SignatureAlgorithm::Es256),
    value: Some(digest),
    ..Default::default()
};

let result = key_client
    .sign("cert-name", "", sign_params.try_into()?, None)
    .await?
    .into_model()?;
println!("Signature: {:?}", result.result);
```

## Certificate Formats

| Format | Content Type | Use Case |
| ------ | ------------ | -------- |
| PKCS#12 | `application/x-pkcs12` | Bundled cert + private key |
| PEM     | `application/x-pem-file` | Base64-encoded, common in Linux/web |

## Best Practices

1. **Use Entra ID auth** — `DeveloperToolsCredential` for dev, `ManagedIdentityCredential` for production
2. **Use `..Default::default()`** — for struct update syntax on all model types
3. **Use `ResourceExt`** — to extract certificate name/version from IDs
4. **Poll LROs** — `create_certificate` returns a `Poller`; call `.wait().await` for completion
5. **Reuse clients** — `CertificateClient` is thread-safe

## Reference Links

| Resource      | Link                                                              |
| ------------- | ----------------------------------------------------------------- |
| API Reference | https://docs.rs/azure_security_keyvault_certificates              |
| crates.io     | https://crates.io/crates/azure_security_keyvault_certificates     |
