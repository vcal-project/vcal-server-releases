# VCAL Server v0.6.0

This is the first public GitHub Release for VCAL Server.

VCAL Server is commercial, closed-source software. This repository does not contain the VCAL Server source code.

## Downloads

The following Linux builds are attached to this release:

- `vcal-server-linux-x86_64-v0.6.0.tar.gz`
- `vcal-server-linux-x86_64-musl-v0.6.0.tar.gz`
- `vcal-server-linux-aarch64-musl-v0.6.0.tar.gz`

Each archive is published together with:

- `.sha256` checksum file
- `.minisig` signature file

The same release files are also available through the official VCAL Server download page:

https://vcal-project.com/vcal-server/

Full VCAL Server documentation is available here: https://server.docs.vcal-project.com/

## Build selection

Use:

- `linux-x86_64` for standard Linux x86_64 environments
- `linux-x86_64-musl` for portable/static-style Linux x86_64 deployments
- `linux-aarch64-musl` for ARM64 Linux deployments

## Verification

After downloading an archive and its checksum file:

```bash
sha256sum -c vcal-server-linux-x86_64-v0.6.0.tar.gz.sha256
```

Replace the filename with the build you downloaded.

## API Documentation

The static OpenAPI specification is provided as a release artifact where applicable.

The standard free testing build of VCAL Server does not expose the built-in OpenAPI/docs UI at runtime. The docs UI is disabled by default in the binary build.

For normal use, refer to the public documentation and the static `openapi.yml` file: https://server.docs.vcal-project.com/

Customers who need runtime OpenAPI/docs UI access can request a custom build through VCAL Server support.

https://server.docs.vcal-project.com/

## Support

For licensing, support, or product questions, use the official contact channel:

https://vcal-project.com/vcal-server/#contact
