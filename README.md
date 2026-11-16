<p align="center">
  <img src="https://raw.githubusercontent.com/storytold/pdfcraft/main/assets/app-icon/hicolor/256x256/apps/ai.storyteller.pdfcraft.png" width="128" height="128" alt="PdfCraft application icon">
</p>

<h1 align="center">PdfCraft Web — Docker Image</h1>
<p align="center">
  A community-maintained Docker image for
  <a href="https://github.com/storytold/pdfcraft">PdfCraft</a>,
  bringing its Rust and WebAssembly-based PDF workbench to a self-hosted web interface.
  <br>
  Automated builds and image publication to GitHub Container Registry (GHCR).
</p>

<p align="center">
  <a href="https://github.com/MikaPST/pdfcraft-web/actions/workflows/publish-image.yml">
    <img src="https://github.com/MikaPST/pdfcraft-web/actions/workflows/publish-image.yml/badge.svg" alt="Build and publish PdfCraft Web">
  </a>
  <a href="https://github.com/MikaPST/pdfcraft-web/releases/latest">
    <img src="https://img.shields.io/github/v/release/MikaPST/pdfcraft-web?label=Release&logo=github&color=blue" alt="Latest Release">
  </a>
  <a href="https://github.com/MikaPST/pdfcraft-web/pkgs/container/pdfcraft-web">
    <img src="https://ghcr-badge.egpl.dev/mikapst/pdfcraft-web/latest_tag?trim=major&label=GHCR" alt="GHCR image tag">
  </a>
  <a href="docker-compose.yml">
    <img src="https://img.shields.io/badge/Docker%20Compose-ready-2496ED?logo=docker&logoColor=white" alt="Docker Compose configuration">
  </a>
</p>

---

## Why this repository?

The official PdfCraft project provides a WebAssembly-based PDF workbench that can be served as a static website. This repository builds the **official upstream web application** and packages its compiled files into a lightweight, non-root NGINX Docker image for straightforward self-hosting.

The goals are to:

- Build PdfCraft Web from a specific official upstream release tag.
- Publish ready-to-run Docker images to GHCR for `linux/amd64` and `linux/arm64`.
- Automatically detect new official PdfCraft releases and support manual or forced rebuilds.
- Publish a matching GitHub Release **only after the multi-platform Docker image is successfully pushed**.
- Include shared fonts from [storytold/craft-fonts](https://github.com/storytold/craft-fonts).
- **Download and verify the OCR models**, which the standard Trunk compilation does not include.
- Provide Docker Compose and an independently maintained Unraid Community Applications template.
- Maintain independent, version-controlled `Dockerfile`, `nginx.conf`, `.dockerignore` and GitHub Actions files without modifying upstream source.

This is independent community packaging, not an official PdfCraft release or a fork of the application. It is not affiliated with or endorsed by the PdfCraft or ArtCraft maintainers.

## How it works

PdfCraft is a PDF workbench written in Rust. Its browser edition uses WebAssembly, and NGINX serves compiled static files. The container is **not** a remote desktop, PDF processing server or online document-storage service.

- **Browser-based PDF workbench:** read, search, organize, combine, split, annotate, fill forms and edit documents using the web interface, subject to upstream feature availability.
- **Local document processing:** PDF operations run in the browser. The container does not receive files for server-side processing.
- **Local-first upstream design:** the application is built to work without accounts or cloud processing; review upstream documentation for the current behaviour of individual features.
- **OCR:** text recognition uses browser-side WebAssembly workers and ONNX model files delivered by the same web server.
- **GPU support:** WebGPU when available; WebGL2 and CPU fallback options are provided.
- **Self-hosted:** serve the application on your LAN or through a secure HTTPS reverse proxy.

Some native-only functions are not available in the browser edition. Consult the [upstream web documentation](https://github.com/storytold/pdfcraft/blob/main/packaging/web/README.md) for current details.

PdfCraft is **alpha software**. Test updates before relying on it for important documents, and keep independent copies of your PDFs.

## Docker image

The published Docker image is:

```text
ghcr.io/mikapst/pdfcraft-web:latest
```

Successfully packaged upstream releases also receive their matching official version tag, for example:

```text
ghcr.io/mikapst/pdfcraft-web:v0.21.1
```

Use a versioned tag to stay on a particular upstream release. The `latest` tag is mutable, and even version tags can be replaced by an explicitly forced rebuild. For strict reproducibility, pin the published image digest.

### Option 1 — Docker Compose (recommended)

A `docker-compose.yml` file is provided at the root of this repository:

```yaml
services:
  pdfcraft-web:
    image: ghcr.io/mikapst/pdfcraft-web:latest
    container_name: pdfcraft-web
    ports:
      - "8080:8080"
    restart: unless-stopped
```

From the repository root, start the application:

```bash
docker compose up -d
```

Open the application in your browser:

```text
http://YOUR_SERVER_IP:8080
```

Replace `YOUR_SERVER_IP` with your server's IP address or hostname. On the Docker host, you can use `http://localhost:8080`.

#### Useful Compose commands

```bash
# View container logs
docker compose logs -f

# Pull the latest image and recreate the container
docker compose pull
docker compose up -d

# Stop and remove the container
docker compose down
```

No `.env` file or persistent application volume is required for the default deployment. The container serves the application and OCR resources as static files; document and application state is handled by the browser.

To change the host port, edit the left-hand side of `8080:8080`. For example, `8081:8080` publishes the application on port `8081` of the host.

### Option 2 — Run directly with Docker

If you prefer not to use Docker Compose:

```bash
docker pull ghcr.io/mikapst/pdfcraft-web:latest

docker run -d \
  --name pdfcraft-web \
  --restart unless-stopped \
  -p 8080:8080 \
  ghcr.io/mikapst/pdfcraft-web:latest
```

Open `http://YOUR_SERVER_IP:8080`, or `http://localhost:8080` from the Docker host.

Optional resource limits can be added to `docker run`, such as `--cpus=2 --memory=2g`. These limits are not part of the default Compose file; most PDF processing work happens in the client browser.

### Network access and reverse proxies

The runtime container listens on **port `8080`** using NGINX, with the application running as the unprivileged `nginx` user. For remote access, use an HTTPS reverse proxy such as Traefik.

The version-controlled [`nginx.conf`](nginx.conf) provides:

- `application/wasm` for `.wasm` and `text/javascript` for `.js` files.
- Gzip compression for WebAssembly, JavaScript and other static resources.
- Long-lived immutable caching for content-hashed `.wasm` and `.js` files.
- `Cache-Control: no-cache` for `index.html` and for non-hashed OCR model files under `models/`.
- `Cross-Origin-Opener-Policy: same-origin`, as recommended by the upstream hosting guide, and `X-Content-Type-Options: nosniff`.
- A lightweight `GET /healthz` endpoint and Docker health check.

**HTTPS is recommended.** WebGPU and browser clipboard features require a secure context (`https://` or `http://localhost`). Over ordinary HTTP network addresses, PdfCraft can fall back to WebGL2.

PdfCraft does not currently require `Cross-Origin-Embedder-Policy` (COEP) for `SharedArrayBuffer`. If a proxy or hosting platform enforces `COEP: require-corp` globally, follow the upstream guidance for the appropriate `Cross-Origin-Resource-Policy` header.

Upstream uses relative asset URLs (`public_url = "./"`), allowing deployment at a domain root or under a URL prefix **when the reverse proxy maps the prefix correctly to the application's root**. Do not expose an unprotected HTTP port directly to the public Internet.

### Opening a PDF from a URL

PdfCraft supports a `file` query parameter to request a PDF when it starts:

```text
https://YOUR_DOMAIN/?file=https%3A%2F%2Fexample.com%2Fdocument.pdf
```

URL-encode the document URL. The PDF must be hosted on the same origin or allow PdfCraft's origin through CORS. This does not turn the container into a proxy or server-side document store.

### Renderer options

The browser edition selects a renderer automatically. Query flags provide fallback choices:

| URL option | Behaviour |
|---|---|
| No option | WebGPU where available, otherwise WebGL2 |
| `?webgl` | Force WebGL2, useful for WebGPU compatibility problems |
| `?cpu` | Force the CPU canvas path; slower but more broadly compatible |

For example:

```text
https://YOUR_DOMAIN/?webgl
```

### OCR models and browser workers

PdfCraft's **Scan & OCR → Recognize Text** feature uses OCR worker scripts and model files located under the site's `models/` directory.

The standard `trunk build --release` does **not** download these models. This packaging workflow runs the official command `cargo xtask models DEST` after compilation to obtain and verify the models against the upstream `ATTRIBUTION.toml` SHA-256 values. Model licences and attribution are included alongside the models.

- The OCR assets are **bundled into the Docker image**, so they do not need to be fetched from the original model hosts at runtime.
- The browser downloads the required models from your own PdfCraft instance when OCR is first used.
- Model files are not content-hashed; NGINX requires browser cache revalidation for `/models/`.
- The WebAssembly OCR worker assets are produced by the upstream Trunk build, whose post-build hook assigns content-hashed names.
- OCR work consumes memory on the **client**, not on the NGINX server. Upstream notes estimate around 600 MB per active worker.

OCR model licenses are independent of the application's MIT/Apache licensing; see the published `models/` attribution files and the [upstream hosting guide](https://github.com/storytold/pdfcraft/blob/main/packaging/web/README.md).

## Repository structure

The repository maintains its Docker configuration independently of the official PdfCraft application source:

```text
pdfcraft-web/
├── .github/
│   └── workflows/
│       └── publish-image.yml
├── .dockerignore
├── Dockerfile
├── nginx.conf
├── docker-compose.yml
└── README.md
```

| File | Purpose |
|---|---|
| `Dockerfile` | Builds the non-root NGINX runtime image from already compiled static assets. |
| `nginx.conf` | Defines port 8080, WebAssembly MIME types, compression, caching, security headers and `/healthz`. |
| `.dockerignore` | Keeps generated Docker build contexts clean and excludes unnecessary files. |
| `docker-compose.yml` | Provides a ready-to-use Docker Compose deployment. |
| `.github/workflows/publish-image.yml` | Detects official releases, compiles Rust/Trunk, bundles and verifies OCR models, publishes multi-platform images and creates GitHub Releases. |
| `README.md` | Documents deployment, architecture, automation, licence notices and upstream resources. |

### Docker build architecture

**PdfCraft compiles in GitHub Actions, before the Docker image is built.** Unlike PhotoCraft's multi-stage Dockerfile, the PdfCraft Dockerfile contains only the runtime packaging stage; it does not compile Rust for each CPU architecture.

```text
GitHub Actions workspace/
├── packaging/                # MikaPST/pdfcraft-web
│   ├── Dockerfile            # version-controlled, not generated inline
│   ├── nginx.conf
│   ├── .dockerignore
│   └── docker-compose.yml
├── upstream/                 # storytold/pdfcraft (official release tag)
├── craft-fonts/              # storytold/craft-fonts (main)
└── docker-build/             # temporary Docker context
    ├── Dockerfile            # copied from packaging/
    ├── nginx.conf            # copied from packaging/
    ├── .dockerignore         # copied from packaging/
    └── dist/                 # upstream/dist/web plus auxiliary files
        ├── index.html
        ├── models/           # SHA-256-verified OCR models and notices
        ├── licenses/         # upstream and font licence notices
        └── ...               # WebAssembly, JS and other static assets
```

The build process is:

1. Resolve the latest official PdfCraft release or the specified `upstream_tag`.
2. Checkout the **exact official release tag** into `upstream/`, and separately checkout `craft-fonts` from `main`.
3. Record the full PdfCraft and shared-fonts source commits for traceability.
4. Install Rust, the `wasm32-unknown-unknown` target, Trunk and the `wasm-bindgen-cli` version matching upstream `Cargo.lock`.
5. Run `trunk build --release` from `upstream/apps/pdfcraft-web` using `CRAFT_FONTS_DIR`.
6. Validate `upstream/dist/web/index.html`, generated `.wasm` / `.js` files, and relative HTML asset URLs.
7. Copy the static site into `docker-build/dist/` and preserve upstream and shared-font licence notices.
8. Run `cargo xtask models` to fetch and verify OCR assets and their licensing information into `docker-build/dist/models/`.
9. Copy the versioned root `Dockerfile`, `nginx.conf` and `.dockerignore` into `docker-build/`.
10. Use Docker Buildx to publish the same static site for `linux/amd64` and `linux/arm64`.
11. Create or update the matching GitHub Release **after the GHCR push succeeds**.

The `docker-build/`, `upstream/` and `craft-fonts/` checkout directories are temporary CI working directories; they are **not** committed to this repository.

The final image does not contain a Rust compiler or Trunk and needs no database, GPU passthrough, persistent server-side volume or desktop environment.

> [!NOTE]
> The runtime image packages one upstream application release, but the separately checked-out `craft-fonts` repository follows `main`. Its precise commit is recorded in the image metadata and GitHub Release for traceability. A rebuild can therefore contain updated font assets even if the PdfCraft release tag is unchanged.

## Automated builds

GitHub Actions checks the official [PdfCraft releases](https://github.com/storytold/pdfcraft/releases), builds the selected upstream tag and publishes images to GHCR.

| Trigger | Behaviour |
|---|---|
| Daily schedule (`03:47 UTC`) | Checks for the latest official PdfCraft release and skips versions already published here. |
| Manual workflow dispatch | Builds the latest official release or a specified `upstream_tag`. |
| Force rebuild | Rebuilds an existing upstream version with `force_rebuild: true`. |
| Packaging changes on `main` | Rebuilds the current latest official release after changes to the workflow, `Dockerfile`, `nginx.conf`, `.dockerignore`, `docker-compose.yml` or `README.md`. |

### Published tags

Each successful build publishes the following GHCR tags:

- `latest` — the most recently successfully packaged upstream release.
- `<upstream-version>` — the exact official release tag, for example `v0.21.1`.
- `upstream-<commit>` — the first 12 characters of the PdfCraft source commit.

Both `linux/amd64` and `linux/arm64` are supported. The WebAssembly app and OCR assets are assembled once on the GitHub Actions runner; Docker Buildx packages the same static output for each architecture.

### GitHub Releases

After a successful multi-platform Docker build and GHCR push, the workflow creates or updates a [GitHub Release](https://github.com/MikaPST/pdfcraft-web/releases) matching the official upstream version.

Each release includes:

- The official upstream release name, source link and publication date.
- Official PdfCraft release notes.
- The full PdfCraft and `craft-fonts` source commits.
- Published Docker tags and supported architectures.
- The published Docker image digest.
- A note confirming that the OCR models were downloaded and SHA-256-verified during CI.
- An explicit notice that this is independent community packaging.

The Release badge at the top of the README reflects the latest published GitHub Release.

Scheduled and ordinary manual runs skip an upstream version that already has a release here. To rebuild it, open **Actions → Build and publish PdfCraft Web → Run workflow** and enable `force_rebuild`. Changes to monitored packaging files on `main` also trigger a rebuild.

> [!IMPORTANT]
> GitHub Releases are created or updated **only after** successful image publication to GHCR. The `latest` tag is mutable, and forced rebuilds can also update versioned tags. For a fully pinned deployment, use the published image digest.

## Unraid Community Applications

PdfCraft Web can be deployed through the Unraid Community Applications template maintained separately in [MikaPST/unraid-community-apps](https://github.com/MikaPST/unraid-community-apps).

- **Unraid Community Applications repository:** [MikaPST/unraid-community-apps](https://github.com/MikaPST/unraid-community-apps)
- **PdfCraft Web template:** [pdfcraft-web.xml](https://github.com/MikaPST/unraid-community-apps/blob/main/templates/pdfcraft-web.xml)
- **Container image:** `ghcr.io/mikapst/pdfcraft-web:latest`
- **Container port:** `8080`
- **Default host port:** `8080`

The XML template is maintained independently of this image-build repository. Docker Compose and direct Docker deployment remain available alternatives.

## Upstream project

PdfCraft is a Rust-based PDF reader and editor developed by the ArtCraft Team and contributors.

- **Official source:** [storytold/pdfcraft](https://github.com/storytold/pdfcraft)
- **Official website:** [getartcraft.com/apps/pdfcraft](https://getartcraft.com/apps/pdfcraft)
- **Official hosting guide:** [packaging/web/README.md](https://github.com/storytold/pdfcraft/blob/main/packaging/web/README.md)
- **Official web build configuration:** [apps/pdfcraft-web/Trunk.toml](https://github.com/storytold/pdfcraft/blob/main/apps/pdfcraft-web/Trunk.toml)
- **Official development documentation:** [docs/development.md](https://github.com/storytold/pdfcraft/blob/main/docs/development.md)
- **Shared fonts:** [storytold/craft-fonts](https://github.com/storytold/craft-fonts)
- **Official releases:** [PdfCraft releases](https://github.com/storytold/pdfcraft/releases)

The MikaPST GitHub Release tracks the upstream PdfCraft version but represents independent Docker packaging, not an official PdfCraft distribution.

## License and attribution

PdfCraft is dual-licensed under the **MIT License or Apache License 2.0**, at the user's option. Copyright belongs to its upstream authors and contributors.

The packaging process preserves the upstream `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE`, `ATTRIBUTION.md` and `ATTRIBUTION.toml` files under the published site's `/licenses/` directory. Available shared-font licence notices are also included. OCR models have their own licences, attribution and SHA-256 declarations, which are staged alongside the model files under `/models/`.

- [Upstream MIT License](https://github.com/storytold/pdfcraft/blob/main/LICENSE-MIT)
- [Upstream Apache License 2.0](https://github.com/storytold/pdfcraft/blob/main/LICENSE-APACHE)
- [Upstream NOTICE](https://github.com/storytold/pdfcraft/blob/main/NOTICE)
- [Upstream asset attributions](https://github.com/storytold/pdfcraft/blob/main/ATTRIBUTION.md)
- [Shared-font sources and notices](https://github.com/storytold/craft-fonts)

The upstream application's licenses do not automatically license this repository's independent packaging files. Consult any separately provided packaging `LICENSE` file for the terms governing those files.

The PdfCraft and ArtCraft names, logos and other branding remain subject to upstream trademark terms; see [docs/brand/LICENSE-brand.txt](https://github.com/storytold/pdfcraft/blob/main/docs/brand/LICENSE-brand.txt).

## Disclaimer

PdfCraft is alpha software. Features, browser compatibility and behaviour may change between releases. Test image updates and keep independent backups of important PDF documents.

This repository is independently maintained and is not affiliated with, sponsored by or endorsed by the official PdfCraft or ArtCraft maintainers.
