# Seerene Analysis Toolchain for Docker

`ghcr.io/seerene/seerene` is the base image for running the Seerene analysis toolchain inside your own
network. It crawls your code repositories, ticket systems and third-party metric tools (e.g. SonarQube),
runs the analyses and uploads the results to the Seerene platform. Source code never leaves your network.

```sh
docker pull ghcr.io/seerene/seerene:latest
docker run --rm ghcr.io/seerene/seerene:latest print-version
```

Credentials for the systems to be analyzed are kept in an encrypted credentials store, created with the
[Seerene Credentials Store utility](https://github.com/Seerene/credentials-store).

## Seerene Base Image

| | |
|---|---|
| Base | Debian 12 (`bookworm-slim`) |
| Architectures | `linux/amd64`, `linux/arm64` |
| Executables on `PATH` | `analysis_tool_orchestrator` (entry point), `localanalyzer`, `thirdparty_codemetrics_converter` |
| Also included | `git`, `jq`, CA certificates |
| Working directory | `/seerene` |

Seerene rebuilds the image at least twice a month with current Debian packages. Besides `latest`, every
build is tagged with its date (e.g. `2026.10.15`), which can be used to pin a specific version.
Single-architecture images are tagged `<date>-amd64` and `<date>-arm64`.

## Folder Structure

The Seerene bundle consists of "Input Artifacts" and "Work Artifacts". Input Artifacts
are the configurations of the analysis toolchain. Work Artifacts contain data related to
code repositories or tickets and the intermediate steps of the analysis.

### Directories for Input Artifacts

* **configuration**: Bundle-specific configurations for code repositories, third-party tools,
  credentials (`.credentials_store`) and lifecycle management (`orchestration.json`, `ticket_config/`).

### Directories for Work Artifacts

* **author_deduplication**: Iterative file for author deduplication. Developers often use different Git
  identifiers, and the tooling recognizes and deduplicates these. Must be persistent.
* **workspace**: Data related to code repositories, third-party metric tools, and the corresponding
  intermediate steps of the analysis.
* **ticket_data**: Data related to ticket projects and tickets, and the corresponding intermediate steps
  of the analysis.
* **third_party_metrics** (optional input): external metrics per repository.
* **log_reports** (optional output): logs of each run.

As a best practice, Input Artifacts are placed in the container at build time and Work Artifacts are
located on persistent storage that is mounted into the container at deployment time.

### Folder Structure and Mounts in the Image

The image expects this structure below `/seerene`:

```
/seerene
|-- configuration/           Input Artifact: orchestration.json, .credentials_store, ticket_config/
|-- workspace/               Work Artifact: checked-out repositories and analysis results
|-- ticket_data/             Work Artifact: crawled, fetched and analyzed tickets
|-- author_deduplication/    Work Artifact: author_deduplication.json (must be persistent)
|-- third_party_metrics/     optional input: external metrics per repository
`-- log_reports/             optional output: logs of each run
```

These directories are declared as volumes in the image. Mount persistent storage on the Work Artifact
directories (`workspace`, `ticket_data`, `author_deduplication`) and, if used, on `third_party_metrics`
and `log_reports`. A directory that is not mounted ends up in an anonymous Docker volume that is lost
when the container is removed. `configuration` is either added to the image at build time (see below) or
mounted as well; mounting it is required if ticket project crawling is active, because the toolchain
extends the configuration during the run. Mounting the whole `/seerene` directory from one persistent
location is an equivalent shortcut.

### Write Access Requirements

The toolchain needs write access to the working directory `/seerene` and all directories listed
above, as well as to the home directory of the user running it.

## Setting up the Credentials Store

The Seerene analysis toolchain uses a shared credentials store to manage the access information for
the systems it analyzes. The store is an encrypted file named `.credentials_store`, generated and
managed by the `credentials_store` executable from
[github.com/Seerene/credentials-store](https://github.com/Seerene/credentials-store). Download the
binary for your platform from its
[Releases](https://github.com/Seerene/credentials-store/releases) page.

Most systems such as GitHub, GitLab, Bitbucket, Jira or SonarQube authenticate with personal access
tokens, sometimes combined with a username. If authentication is by token only, set the username to
`token` and put the token into the password entry.

For Git, two kinds of credentials are stored: **Git API credentials** for the code hosting platform's
REST API, and **Git clone credentials** for cloning the repositories (HTTPS URLs only). They can be
identical for the same user.

The credentials are passed via a `credentials_config.json` with a `key`, `username` and `password`
per entry. The keys must match the names used in the analysis configuration files:

```json
[
    {"key": "git_api_credentials",   "username": "<username>", "password": "<git-personal-access-token-api>"},
    {"key": "git_clone_credentials", "username": "<username>", "password": "<git-personal-access-token-clone>"},
    {"key": "jira_credentials",      "username": "token",      "password": "<jira-personal-access-token>"},
    {"key": "sonarqube_credentials", "username": "token",      "password": "<sonarqube-user-token>"}
]
```

Create the store in the bundle directory, verify it and move it into `configuration`:

```sh
credentials_store add-json --json credentials_config.json
credentials_store check --json credentials_config.json
mv .credentials_store configuration/
```

## Seerene Dockerfile

Seerene provides an example Dockerfile based on the base image, which you enrich with optional
certificates (e.g. for internal code or ticket management systems, or for proxy communication) and the
`configuration` directory. The image is built in your environment. Required access:

* **Base Image**: the GitHub Container Registry at [ghcr.io](https://ghcr.io).
* **Dependencies**: Debian package repositories, only if additional packages are installed in the Dockerfile.

### Best Practice

Keep all Input Artifacts, i.e. the `configuration` directory including the credentials store and the
Dockerfile, in a Git repository.

### Directory Structure

```
/working_directory
|-- Dockerfile
|-- docker-compose.yml
|-- bundle
    |-- configuration
```

### Sample Dockerfile

```dockerfile
# Sample customer image on top of the Seerene base image.
# Build from this directory: docker build -t my-seerene .
FROM ghcr.io/seerene/seerene:latest

# Optional: trust a corporate CA for Git, Jira, SonarQube or the Seerene platform.
# COPY corporate-ca.crt /usr/local/share/ca-certificates/
# RUN update-ca-certificates

# Optional: bake the configuration into the image instead of mounting it at runtime.
COPY bundle/configuration/ /seerene/configuration/

# Optional: change the default command (see `analysis_tool_orchestrator --help`).
CMD ["execute-all"]
```

### Building the Docker Image

```sh
docker build -t my-seerene .
```

If your environment uses an HTTP/HTTPS proxy, pass the proxy settings as build arguments:

```sh
docker build \
  --build-arg http_proxy=http://proxy.example.com:8080 \
  --build-arg https_proxy=http://proxy.example.com:8080 \
  -t my-seerene .
```

## Deploying the Docker Container

The entry point of the image is `analysis_tool_orchestrator`; the command to run is passed as argument:

| Command | What it does |
|---|---|
| `print-version` | prints the orchestrator version (default command) |
| `probe-all` | probe only, no upload; result in `workspace/probe_results-<timestamp>.zip` |
| `execute-all` | full run: crawl, analyze, upload to the Seerene platform |

Both `probe-all` and `execute-all` accept `--analysis-group-key` and `--since`; see
`docker run --rm ghcr.io/seerene/seerene:latest execute-all --help`.

### Docker Compose

The sample `docker-compose.yml` builds the image from the Dockerfile above and mounts the persistent
directories from the working directory:

```yaml
services:
  seerene:
    build: .
    image: my-seerene
    command: ["execute-all"]
    volumes:
      - ./bundle/configuration:/seerene/configuration
      - ./workspace:/seerene/workspace
      - ./ticket_data:/seerene/ticket_data
      - ./author_deduplication:/seerene/author_deduplication
      - ./third_party_metrics:/seerene/third_party_metrics
      - ./log_reports:/seerene/log_reports
```

```sh
docker compose run --build --rm seerene              # execute-all
docker compose run --build --rm seerene probe-all    # probe only
```

`--build` rebuilds the image when the Dockerfile or the baked-in configuration changed.

### Kubernetes

A Deployment starts a pod with the container. The directories `author_deduplication`, `workspace` and
`ticket_data` are mounted into `/seerene` from Persistent Volume Claims with `ReadWriteMany` access;
add `third_party_metrics` and `log_reports` in the same way if used. CPU and memory limits can be
specified as needed.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: run-job
spec:
  replicas: 1
  selector:
    matchLabels:
      app: run-job
  template:
    metadata:
      labels:
        app: run-job
    spec:
      containers:
      - name: run-container
        image: <your-image-here>
        args: ["execute-all"]
        volumeMounts:
        - name: author_deduplication
          mountPath: /seerene/author_deduplication
        - name: ticket_data
          mountPath: /seerene/ticket_data
        - name: workspace
          mountPath: /seerene/workspace
      restartPolicy: Always
      volumes:
      - name: author_deduplication
        persistentVolumeClaim:
          claimName: author_deduplication-pvc
      - name: ticket_data
        persistentVolumeClaim:
          claimName: ticket_data-pvc
      - name: workspace
        persistentVolumeClaim:
          claimName: workspace-pvc
```

Each claim requests storage from a file share with `ReadWriteMany` access mode, for example on Azure
File Storage (`storageClassName: azurefile`). Size `workspace` according to the amount of source code
to be analyzed.

## Network Access

The execution environment needs network access to the systems with the data to be analyzed. If the
analysis results are to be uploaded automatically, HTTPS access to `tp.seerene.com` must be allowed.

## Getting Configuration

`orchestration.json`, ticket configurations and the tokens for your Seerene platform are provided by
your Seerene contact.
