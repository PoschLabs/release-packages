# release-packages

Reusable workflow that signs and publishes Linux packages. It signs the RPMs
with [`sign-rpms`](https://github.com/PoschLabs/sign-rpms), then publishes the
signed RPMs and the DEBs with
[`publish-packages`](https://github.com/PoschLabs/publish-packages).

## Usage

```yaml
jobs:
  build:
    # ... uploads artifact "unsigned-linux-packages"

  release:
    needs: build
    permissions:
      contents: read
      id-token: write
    uses: PoschLabs/release-packages/.github/workflows/release.yml@v1
    with:
      package-server-url: ${{ vars.PACKAGE_SERVER_URL }}
    secrets:
      package-server-username: ${{ secrets.PACKAGE_SERVER_USERNAME }}
      package-server-password: ${{ secrets.PACKAGE_SERVER_PASSWORD }}
      sign-infisical-url: ${{ secrets.SIGN_INFISICAL_URL }}
      sign-infisical-identity-id: ${{ secrets.SIGN_INFISICAL_IDENTITY_ID }}
      sign-infisical-project-id: ${{ secrets.SIGN_INFISICAL_PROJECT_ID }}
      sign-infisical-secret-path: ${{ secrets.SIGN_INFISICAL_SECRET_PATH }}
```

The secrets are not inherited: the caller must map each one explicitly, as
shown above.

## How it works

1. **sign**: downloads the artifact `artifact-name`, signs every `*.rpm` in its
   top directory and uploads the result as `signed-rpm-packages`.
2. **publish-rpm**: publishes `signed-rpm-packages` to `rpm-repository`
   (skipped if `publish-rpm` is `false`).
3. **publish-deb**: publishes the DEBs from the unsigned `artifact-name`
   artifact to `deb-repository` (skipped if `publish-deb` is `false`). DEBs are
   not signed by this workflow.

The publish jobs run only after signing succeeds, even when `publish-rpm` is
`false`.

## Inputs

| Input | Default | Description |
|---|---|---|
| `package-server-url` | *required* | URL of the package server |
| `artifact-name` | `unsigned-linux-packages` | Artifact with the unsigned packages |
| `rpm-repository` | `yum` | Target RPM repository |
| `deb-repository` | `apt` | Target DEB repository |
| `publish-rpm` | `true` | Run the RPM publish job |
| `publish-deb` | `true` | Run the DEB publish job |

## Secrets

All secrets are required. The caller's secret is the one you create in GitHub;
the input name is what you pass it as.

| Input | Caller's secret | Level | Description |
|---|---|---|---|
| `package-server-username` | `PACKAGE_SERVER_USERNAME` | Organisation or repository | Package server user |
| `package-server-password` | `PACKAGE_SERVER_PASSWORD` | Organisation or repository | Package server password |
| `sign-infisical-url` | `SIGN_INFISICAL_URL` | Organisation, shared with the repository | Address of the Infisical instance |
| `sign-infisical-identity-id` | `SIGN_INFISICAL_IDENTITY_ID` | Repository | ID of the repository's own Infisical machine identity (OIDC login) |
| `sign-infisical-project-id` | `SIGN_INFISICAL_PROJECT_ID` | Organisation, shared with the repository | Infisical project that holds the signing key |
| `sign-infisical-secret-path` | `SIGN_INFISICAL_SECRET_PATH` | Organisation, shared with the repository | Folder in that project with the signing key secrets |

The `sign-infisical-*` secrets are forwarded to `sign-rpms` under its
upper-case names (`SIGN_INFISICAL_URL`, ...). They are secrets, not variables,
so they are masked in the logs of public repositories. `sign-rpms` fails early,
naming any that are empty. Callers that used the earlier `SIGN_INFISICAL_*`
variables must create secrets and add the mapping.

## Prerequisites

- The calling job must grant `permissions: contents: read` and
  `id-token: write` (the latter for the OIDC login to Infisical).
- A self-hosted runner in the `package-publishing` group with the `rpm-signer`
  label must be available to the repository.
- The signing key lives in Infisical and the repository needs its own machine
  identity there; see the setup guides in `sign-rpms`.
- `PACKAGE_SERVER_URL` is typically an organisation variable; the package
  server credentials are secrets.
