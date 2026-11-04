<p align="center">
  <img src="https://raw.githubusercontent.com/storytold/pdfcraft/main/assets/app-icon/hicolor/256x256/apps/ai.storyteller.pdfcraft.png" width="128" height="128" alt="PdfCraft application icon">
</p>

<h1 align="center">PdfCraft Web — Docker Image</h1>

<p align="center">
  A community-maintained Docker image for
  <a href="https://github.com/storytold/pdfcraft">PdfCraft</a>,
  bringing its WebAssembly-based PDF reader and editor to a self-hosted web interface.
  <br>
  Automated weekly builds and image publication to GitHub Container Registry (GHCR).
</p>

<p align="center">
  <a href="https://github.com/MikaPST/pdfcraft-web/actions/workflows/publish-image.yml">
    <img src="https://github.com/MikaPST/pdfcraft-web/actions/workflows/publish-image.yml/badge.svg" alt="Build and publish PdfCraft Web">
  </a>
  <a href="https://ghcr.io/mikapst/pdfcraft-web">
    <img src="https://ghcr-badge.egpl.dev/mikapst/pdfcraft-web/latest_tag?trim=major&label=GHCR" alt="GHCR image tag">
  </a>
</p>

---

## Why this repository?

The upstream PdfCraft project provides a WebAssembly build that can be served as a static website. This repository automates building that web application from upstream source and packages the generated files into a lightweight NGINX-based Docker image.

The goals are to:

- Build the web app from the current `main` branch of [storytold/pdfcraft](https://github.com/storytold/pdfcraft).
- Include the Japanese font assets from [storytold/craft-fonts](https://github.com/storytold/craft-fonts).
- Rebuild the image automatically each week and publish it to GHCR.
- Provide a simple image that can later be offered through Unraid Community Applications.
- Keep this packaging project separate from the upstream PdfCraft source code.

This is an independent community packaging project. It is **not an official PdfCraft release** and is not affiliated with or endorsed by the PdfCraft or ArtCraft maintainers.

## How it works

PdfCraft is a PDF workbench written in Rust. Its web application runs in the browser using WebAssembly. The container serves the compiled static web files through NGINX; it does not run a server-side PDF processing service.

- **Browser-based PDF workbench:** read, search, organize, combine, split, annotate, fill forms and work with PDF documents using the web interface.
- **Local document processing:** PDF operations run in the browser. The container only delivers the application files; it does not receive documents for server-side processing.
- **No account or cloud processing:** the upstream project describes PdfCraft as an offline-capable, local-first application without accounts, telemetry or cloud processing.
- **Web-specific limitations:** some native-only capabilities are unavailable in the browser build. Consult the upstream documentation and roadmap for current feature coverage.

PdfCraft is in **alpha** and is under active development. Features, compatibility and behaviour may change between upstream builds. Test updates before relying on the application for important documents.

## Docker image

The published image is:

```text
ghcr.io/mikapst/pdfcraft-web:latest
```

### Quick start with Docker Compose

A `docker-compose.yml` file is provided at the repository root:

```yaml
services:
  pdfcraft-web:
    image: ghcr.io/mikapst/pdfcraft-web:latest
    container_name: pdfcraft-web
    restart: unless-stopped
    ports:
      - "8080:8080"
```

Start PdfCraft Web from the repository directory:

```bash
docker compose up -d
```

Then open:

```text
http://YOUR_SERVER_IP:8080
```

Replace `YOUR_SERVER_IP` with the IP address or hostname of your server. On the same computer, use `http://localhost:8080`.

### Run directly with Docker

```bash
docker run -d \
  --name pdfcraft-web \
  --restart unless-stopped \
  -p 8080:8080 \
  ghcr.io/mikapst/pdfcraft-web:latest
```

### Useful commands

```bash
# View logs
docker compose logs -f

# Pull the latest image and recreate the container
docker compose pull
docker compose up -d

# Stop and remove the container
docker compose down
```

No `.env` file or persistent Docker volume is required for the default configuration. The host port is `8080`; to use a different host port, change the left-hand side of the port mapping, for example `8081:8080`.

### Opening a PDF from a URL

PdfCraft supports the `file` query parameter to open a PDF when the application starts:

```text
https://YOUR_DOMAIN/?file=https%3A%2F%2Fexample.com%2Fdocument.pdf
```

The document URL must be URL-encoded. The PDF must be served from the same origin as PdfCraft or its server must explicitly allow the PdfCraft origin through CORS. This feature does not turn the container into a PDF proxy or document storage service.

### Reverse proxy and browser requirements

The NGINX configuration follows the upstream hosting guidance:

- serves `.wasm` files with the `application/wasm` MIME type;
- serves `.js` files with the `text/javascript` MIME type;
- enables gzip compression for WebAssembly, JavaScript and HTML;
- applies long-lived immutable caching to content-hashed `.wasm` and `.js` assets;
- applies `no-cache` to `index.html`, so the browser checks for a new release.

The upstream build uses relative asset URLs (`public_url = "./"`), so it can be served from a domain root or a URL prefix, provided the reverse proxy preserves the requested path.

**HTTPS is recommended.** WebGPU and clipboard access require a secure browser context (`https://` or `http://localhost`). On other plain-HTTP origins, PdfCraft falls back to WebGL2.

No special COOP or COEP isolation headers are required by PdfCraft's current web build. If your reverse proxy or hosting platform adds COEP `require-corp` globally, follow the upstream hosting notes for the corresponding Cross-Origin-Resource-Policy header.

### Renderer options

PdfCraft selects its renderer automatically:

| URL option | Behaviour |
|---|---|
| No option | WebGPU when available, otherwise WebGL2 |
| `?webgl` | Forces the WebGL2 backend |
| `?cpu` | Forces the CPU canvas path; slowest but most compatible |

For example:

```text
https://YOUR_DOMAIN/?webgl
```

## Automated builds

GitHub Actions builds the PdfCraft web application and publishes the container image automatically.

| Trigger | Behaviour |
|---|---|
| Weekly schedule | Checks out the current upstream `main` branch, builds the WebAssembly app and publishes a new image when the build succeeds |
| Manual workflow dispatch | Allows a build to be started manually from the Actions tab |
| Changes to packaging files | Runs the workflow when `docker-compose.yml`, `README.md` or the workflow file changes on `main` |

### Published tags

- `latest` — the latest successful build.
- `upstream-<commit>` — identifies the exact PdfCraft source commit used for the build.

The image is published for `linux/amd64` and `linux/arm64`. The container serves static web assets through NGINX; the generated web files are architecture-independent.

A scheduled rebuild does not guarantee that upstream changes are compatible or stable. If a build fails, the previous published image remains available.

> **First-time GHCR setup:** after the first successful publication, check the package's visibility in GitHub Packages. Set `pdfcraft-web` to **Public** if you want users to pull the image without authenticating to GHCR.

## Build pipeline

The workflow:

1. Checks out this repository.
2. Checks out PdfCraft and the shared `craft-fonts` repository.
3. Installs the Rust WebAssembly target, Trunk and the `wasm-bindgen-cli` version resolved from the upstream `Cargo.lock`.
4. Runs `trunk build --release` from `upstream/apps/pdfcraft-web`, with `CRAFT_FONTS_DIR` pointing to the checked-out font repository.
5. Uses the upstream Trunk configuration to generate the static site in `upstream/dist/web`.
6. Checks that the generated WebAssembly and JavaScript assets exist and that the HTML uses relative URLs.
7. Packages the static site into an NGINX image and publishes it to GHCR.

The workflow generates the Dockerfile and NGINX configuration during the GitHub Actions run. Separate `Dockerfile`, `nginx.conf` and `.dockerignore` files are therefore not required in this packaging repository.

The upstream web build is based on the same Rust application code as the desktop app, but some native-only features are not available in the browser. Consult the upstream web hosting notes and roadmap when evaluating feature support.

## Unraid Community Applications

This image is intended to be usable on Unraid.

An Unraid Community Applications template will be added after the image build and runtime configuration have been tested and validated.

In the meantime, PdfCraft Web can be deployed on Unraid using the Docker Compose configuration above or by configuring the container manually with:

- **Repository:** `ghcr.io/mikapst/pdfcraft-web:latest`
- **Container port:** `8080`
- **Host port:** `8080` by default

## Upstream project and attribution

- **PdfCraft source code:** [storytold/pdfcraft](https://github.com/storytold/pdfcraft)
- **PdfCraft web hosting instructions:** [packaging/web/README.md](https://github.com/storytold/pdfcraft/blob/main/packaging/web/README.md)
- **PdfCraft development notes:** [docs/development.md](https://github.com/storytold/pdfcraft/blob/main/docs/development.md)
- **Shared font assets:** [storytold/craft-fonts](https://github.com/storytold/craft-fonts)
- **Official releases:** [PdfCraft releases](https://github.com/storytold/pdfcraft/releases)

PdfCraft is dual-licensed under the MIT License or Apache License 2.0, at the user's option, as specified by the upstream project. Required notices are provided in the upstream `NOTICE` file. Bundled fonts, icons, images and other assets retain their own licenses and attribution requirements; consult upstream `ATTRIBUTION.md` and the `craft-fonts` repository.

The ArtCraft name, wordmark and logos in the upstream `docs/brand/` directory are trademarks and are not covered by the software license. Their use is subject to the upstream `docs/brand/LICENSE-brand.txt`. The license of the upstream application and its assets does not automatically determine the license of the packaging files maintained in this repository. If you want to grant reuse rights for your own workflow, Compose file and documentation, select and include a separate `LICENSE` for this repository.

## Disclaimer

This repository is maintained independently and is not affiliated with, sponsored by or endorsed by the PdfCraft project or its maintainers.

PdfCraft is alpha software. Keep independent copies of important documents and test updates before relying on the application for important work.
