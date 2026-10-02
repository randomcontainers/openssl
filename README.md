# OpenSSL

Container images with the `openssl` command from [OpenSSL](https://openssl-library.org/), compiled from the signed release tarball of the newest stable series, currently 4.0. The images are rebuilt for each release of the newest series and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the OpenSSL Project. Report problems with the image in this repository and problems with OpenSSL itself [upstream](https://github.com/openssl/openssl/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/openssl x509 -in cert.pem -noout -text
```

Make a key and a self-signed certificate for `localhost`. Most browsers and TLS libraries ignore the common name and match the host name against `subjectAltName`, so the command sets both:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256 \
  -noenc -keyout key.pem -out cert.pem -days 30 \
  -subj /CN=localhost -addext subjectAltName=DNS:localhost
```

See which TLS version, cipher and key exchange a server negotiates:

```sh
docker run --rm ghcr.io/randomcontainers/openssl s_client -connect example.org:443 -brief
```

The entrypoint runs `openssl` under `tini` in `/work`, so file names are relative to the directory you mount. Without arguments the image runs `openssl help`, which lists the commands. Without `-i`, the container's standard input is empty, so `s_client` disconnects right after the handshake. Pass `-i` to pipe data in (`docker run --rm -i ghcr.io/randomcontainers/openssl dgst -sha256 < file`), and `-it` for an interactive `s_client` session or a passphrase prompt. The [manual pages](https://docs.openssl.org/4.0/man1/) cover every command.

## Post-quantum algorithms and ECH

OpenSSL's default group list puts the hybrid post-quantum group `X25519MLKEM768` first, so `s_client -brief` prints `Negotiated TLS1.3 group: X25519MLKEM768` when a server supports it. Pass `-groups X25519MLKEM768` to offer nothing else. ML-DSA and SLH-DSA keys and certificates work with the usual commands:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/openssl genpkey -algorithm ML-DSA-65 -out mldsa.key
```

OpenSSL 4.0 adds Encrypted Client Hello (ECH, RFC 9849). `openssl ech -public_name example.com -out ech.pem` writes a key pair and its ECHConfigList to `ech.pem`, and `s_server -ech_key ech.pem` loads it. `s_client -ech_config_list` takes the ECHConfigList as one line of base64, so join the lines between `BEGIN ECHCONFIG` and `END ECHCONFIG` before passing it:

```sh
ecl=$(awk '/END ECHCONFIG/ { p = 0 } p { printf "%s", $0 } /BEGIN ECHCONFIG/ { p = 1 }' ech.pem)
docker run --rm ghcr.io/randomcontainers/openssl s_client -connect <host>:<port> -ech_config_list "$ecl"
```

With the line breaks left in, `s_client` stops with `error setting ECHConfigList`. Without `-brief`, it reports the result as `ECH: success: 1`, followed by the inner and outer server names.

## Coming from OpenSSL 3

Ubuntu 26.04 and Alpine 3.24 ship OpenSSL 3.5. These changes in 4.0 affect the command line:

- Engines are gone, and with them the `-engine` options. Use providers instead.
- SSLv3 is removed, and the deprecated elliptic curves of RFC 8422 are no longer available in TLS.
- EC keys with explicit curve parameters fail to load unless the parameters match a named curve.
- `c_rehash` is removed; `openssl rehash` does the same job. `openssl ca` no longer has `-msie-hack`.
- Hex dumps of key data, such as the RSA modulus in `rsa -text`, no longer start with an extra `00:`, and dumps are 16 bytes wide (24 for signatures). Scripts that parse `-text` output may need changes.
- `req`, `x509` and `ca` take the subject and authority key identifiers from the configuration file. The stock `openssl.cnf` in this image adds them.

[NEWS.md](https://github.com/openssl/openssl/blob/openssl-4.0/NEWS.md) lists all incompatible changes in 4.0, including those in the library API.

## What is in the image

- `openssl` in `/usr/local/bin`, with `libssl.so.4` and `libcrypto.so.4` in `/usr/local/lib`.
- The legacy provider, `/usr/local/lib/ossl-modules/legacy.so`, for MD4, RC4, Blowfish, DES-X and other old algorithms. Load it with `-provider legacy -provider default`.
- The OpenSSL directory `/usr/local/ssl` with the stock `openssl.cnf`. `certs` and `cert.pem` in it point to the distro's CA certificates in `/etc/ssl/certs`, so `s_client` and `verify` trust the same CAs as the rest of the system.

The build uses upstream's default options. QUIC, ECH, ML-KEM, ML-DSA and SLH-DSA are in. The FIPS provider, kernel TLS, zlib, Brotli and zstd compression, MD2 and RC5 are not. The headers, static libraries and pkg-config files are left out, and so are the man pages and the `CA.pl` and `tsget` scripts, which need Perl. `openssl version -a` prints the build settings, and the configure options are in `/usr/local/share/randomcontainers/openssl/buildinfo`.

The Ubuntu image also has Ubuntu's own OpenSSL 3, `/usr/bin/openssl` and `libssl.so.3`, because the `ca-certificates` package depends on it. `/usr/local/bin` comes first on the `PATH`, so `openssl` runs this build. The Alpine image has Alpine's `libssl3` and `libcrypto3`, which come with the base image, but no second `openssl` command.

This is an image for the `openssl` command, not a base image for programs that link OpenSSL. Programs installed from distro packages keep loading the distro's `libssl.so.3`.

## Default or slim

OpenSSL's default image adds no other tools, so `latest` and `slim` are the same image: `openssl` and the libraries it links against. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is an OpenSSL release such as `4.0.3`. `<minor>` and `<major>` are its shorter forms, `4.0` and `4`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current OpenSSL version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

Only the newest stable series is built. OpenSSL 4.0 is not a long-term support release: upstream supports it until 2027-05-14, and 4.1 is due before then. When 4.1.0 comes out, `latest`, `4` and the distro tags move to it, and `4.0` stays on the last 4.0 release built here. Later 4.0 patch releases are not built, so an image on the `4.0` tag gets no more security fixes, from OpenSSL or from the distro. Releases of older series, such as the 3.5 long-term support release, are not built at all.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new OpenSSL release and are rebuilt when the base image changes. The packages OpenSSL needs are listed in `/usr/local/share/randomcontainers/openssl/runtime-deps`. Switch to root to change the image, then back. To trust a private CA in `s_client` and `verify`:

```dockerfile
FROM ghcr.io/randomcontainers/openssl:slim-ubuntu@sha256:...
USER root
COPY my-ca.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates
USER 1000:1000
```

The same lines work with `slim-alpine`. The entrypoint is `["tini", "--", "openssl"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new OpenSSL releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/openssl:latest \
  --repo randomcontainers/openssl --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/openssl:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the OpenSSL release signing key in `keys/openssl-release.gpg` (fingerprint `B146 647E 45A7 B339 47AB 226B 2A2C 87D1 6169 2D40`). Releases are signed with a subkey of it that expires on 2027-05-26.

## Updates

The project checks the releases of [openssl/openssl](https://github.com/openssl/openssl/releases) every 15 minutes and skips pre-releases. A release is picked up once it is 24 hours old. Its tarball is checked against the release's `openssl-<version>.tar.gz.sha256` file and the digest GitHub records for the asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. A release signed with a key that is not in `keys/openssl-release.gpg` fails to build until the key is added. OpenSSL usually publishes patch releases of all supported series on the same day, and only the highest version is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t openssl:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

OpenSSL is licensed under the Apache License 2.0, so the image's license label is `Apache-2.0`. OpenSSL's `LICENSE.txt` is in `/usr/local/share/randomcontainers/openssl/licenses/`. The Ubuntu and Alpine packages in the image keep their own licenses.

Every version has a GitHub release in this repository, named `v<version>`, with the exact `openssl-<version>.tar.gz` that was compiled and its signature. The build applies no patches. `/usr/local/share/randomcontainers/openssl/source` lists that release and the upstream download URLs.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
