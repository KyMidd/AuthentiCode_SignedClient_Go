# Authenticode-Signed Go Client Release

A GitHub Actions workflow that builds a Go Windows binary, Authenticode-signs it with an HSM-backed certificate in Azure Key Vault, and publishes it to S3 with a SHA-256 checksum and a `latest.json` manifest.

The build job uses Go. The sign and publish jobs work with any Windows PE binary, so Rust, .NET, C/C++ and other toolchains fit by replacing the build steps and writing the output to `dist/${BINARY_NAME}`.

The workflow is [`.github/workflows/release.yml`](.github/workflows/release.yml). It runs on any pushed tag matching `v*`, and each run is titled with its tag, e.g. "Release v1.2.3".

## Pipeline

| Job | Runner | What it does |
|---|---|---|
| `build` | `ubuntu-latest` | Cross-compiles with `GOOS=windows GOARCH=amd64`, stamps the tag into `main.version`, and uploads the unsigned `.exe` as an artifact |
| `sign` | `windows-latest` | Authenticode-signs the `.exe` through Azure Key Vault, verifies the signature, and uploads the signed `.exe` as an artifact |
| `publish` | `ubuntu-latest` | Computes the SHA-256 of the signed `.exe` and uploads the binary, its `.sha256` and `latest.json` to S3 |

The checksum is computed in `publish`, after signing, so it matches the shipped file.

## Signing

The `sign` job works only with the downloaded artifact and the Azure signing identity.

- **Sign CLI:** installs Microsoft's [Sign CLI](https://github.com/dotnet/sign) at the pinned `SIGN_TOOL_VERSION`. Before installing, it confirms the NuGet package's author and repository match `SIGN_TOOL_EXPECTED_AUTHOR` and `SIGN_TOOL_EXPECTED_REPO`.
- **Azure login:** authenticates with GitHub OIDC through `azure/login`, pinned to a commit SHA, with `allow-no-subscriptions` for an identity scoped to the vault. It logs out with `az logout` in an `if: always()` step.
- **Digest signing:** Sign CLI hashes the `.exe` locally and sends the digest to Key Vault, where the HSM signs it with the private key.
- **Timestamping:** each signature carries an RFC-3161 timestamp from `TIMESTAMP_URL` (DigiCert by default). Windows validates timestamped signatures as of the signing time, so they remain valid after the certificate expires.
- **Verification:** `Get-AuthenticodeSignature` confirms the status is `Valid`, meaning the chain verified against the Windows root store, and that a timestamp is present. When `EXPECTED_CERT_SHA256` is set, the job also compares the SHA-256 of the certificate embedded in the binary against that value. The result is written to the job summary.

## Configuration

### Workflow `env:` block

| Setting | Description |
|---|---|
| `BINARY_NAME` | Output file name, e.g. `my-app.exe` |
| `GO_MAIN_PACKAGE` | Go package to build, e.g. `./cmd/my-app` |
| `PUBLISHER_NAME` | Publisher name embedded in the signature |
| `DESCRIPTION` | Description embedded in the signature |
| `DESCRIPTION_URL` | URL embedded in the signature |
| `SIGN_TOOL_VERSION` | Exact Sign CLI version |
| `SIGN_TOOL_EXPECTED_AUTHOR` | Expected NuGet author for Sign CLI |
| `SIGN_TOOL_EXPECTED_REPO` | Expected NuGet repository URL for Sign CLI |
| `DOTNET_VERSION` | .NET SDK used to run Sign CLI |
| `TIMESTAMP_URL` | RFC-3161 timestamp authority |

### Repository or environment variables

| Variable | Description |
|---|---|
| `AZURE_CLIENT_ID` | Entra app or managed identity client ID |
| `AZURE_TENANT_ID` | Entra tenant ID |
| `AZURE_KEY_VAULT_URL` | Key Vault URL, e.g. `https://my-vault.vault.azure.net/` |
| `AZURE_CERT_NAME` | Name of the code signing certificate in the vault |
| `AWS_ROLE_ARN` | IAM role the `publish` job assumes via OIDC |
| `AWS_REGION` | Bucket region |
| `S3_BUCKET` | Destination bucket |
| `EXPECTED_CERT_SHA256` | Optional. Lowercase hex SHA-256 of the DER signing certificate |

### Azure

1. A Key Vault on the Premium SKU (HSM-backed keys) holding an Authenticode code signing certificate.
2. An Entra app registration or user-assigned managed identity with a federated credential for the repo. To scope signing to a GitHub environment, uncomment `environment: release` on the `sign` job and use the subject `repo:<owner>/<repo>:environment:release`.
3. The **Key Vault Crypto User** and **Key Vault Certificate User** roles for that identity on the vault, or `sign` and `get` under access policies.

### AWS

An IAM role that trusts GitHub's OIDC provider for the repo and allows `s3:PutObject` on `releases/*` in the bucket.

## S3 layout

```
releases/
  latest.json
  versions/
    v1.2.3/
      my-app.exe
      my-app.exe.sha256
```

`latest.json` is written after the versioned objects:

```json
{"version": "v1.2.3", "sha256": "…", "s3_key": "releases/versions/v1.2.3/my-app.exe", "size_bytes": 1234567}
```

## Releasing

Push a tag:

```bash
git tag v1.2.3
git push origin v1.2.3
```

## License

[MIT](LICENSE)
