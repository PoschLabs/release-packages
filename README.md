# release-packages

Reusable workflow that signs and publishes Linux packages
(sign → publish RPM + DEB). Wraps
[`sign-rpm-action`](https://github.com/PoschLabs/sign-rpm-action) and
[`publish-packages-action`](https://github.com/PoschLabs/publish-packages-action).

## Usage

```yaml
jobs:
  build:
    # ... uploads artifact "unsigned-linux-packages"

  release:
    needs: build
    permissions:
      id-token: write
    uses: PoschLabs/release-packages/.github/workflows/release.yml@v1
    with:
      server-url: ${{ vars.PACKAGE_SERVER_URL }}
    secrets:
      server-username: ${{ secrets.PACKAGE_SERVER_USERNAME }}
      server-password: ${{ secrets.PACKAGE_SERVER_PASSWORD }}
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `server-url` | *required* | Package server URL |
| `artifact-name` | `unsigned-linux-packages` | Unsigned packages artifact |
| `rpm-repository` | `yum` | Target RPM repository |
| `deb-repository` | `apt` | Target DEB repository |
| `publish-rpm` / `publish-deb` | `true` | Toggle each publish job |

Secrets: `server-username`, `server-password` (required).

## Prerequisites

### Signing (`sign-rpm-action`)

The signing job reads four non-secret **Actions variables** from the `vars`
context. They are most convenient as organisation variables shared with every
app repository, but repository-level variables work equally well. They are
plain variables, not secrets, and the workflow fails early naming any that are
missing.

| Variable | Description |
|---|---|
| `SIGN_INFISICAL_URL` | Address of the Infisical instance |
| `SIGN_INFISICAL_IDENTITY_ID` | ID of the machine identity used for OIDC login |
| `SIGN_INFISICAL_PROJECT_ID` | Infisical project holding the signing key |
| `SIGN_INFISICAL_SECRET_PATH` | Folder in that project with the signing key secrets |

Variables scoped to a deployment *environment* are only visible to jobs that
declare that `environment:`, which the signing job does not, so define them at
organisation or repository level.

The signing key itself lives in Infisical, so no signing secrets are needed in
GitHub. Also required:

- The calling job must grant `permissions: id-token: write` (OIDC).
- A self-hosted runner in the `package-publishing` group with the `rpm-signer`
  label must be available to the repository.

### Publishing (`publish-packages-action`)

- `server-url` input, typically from the variable `PACKAGE_SERVER_URL`
- Secrets `PACKAGE_SERVER_USERNAME` and `PACKAGE_SERVER_PASSWORD`, passed as
  `server-username` / `server-password`
