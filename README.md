# Jenkins for TerraMaster TOS 7

> TerraMaster TOS 7 App Center — **Docker application** package for the
> [official Jenkins Docker image](https://github.com/jenkinsci/docker).

## What this is

A four-file TOS Docker application archive (`jenkinsdocker.tar.gz`) that installs
the official Jenkins LTS image on TOS 7 through the App Center:

- `config.ini` — TOS application metadata (`application_type: docker`, id `jenkinsdocker`)
- `jenkinsdocker.lang` — 23-language superset (TOS requires 14)
- `jenkinsdocker.svg` — application icon
- `docker-compose.yml` — one service, image `jenkins/jenkins:2.580.2-lts-jdk21`

The container image is used unmodified. The package only supplies the TOS
metadata, the compose file and the icon.

## How it works

| | |
|---|---|
| Container image | `jenkins/jenkins:2.580.2-lts-jdk21` (Docker Hub, multi-arch amd64/arm64, used unmodified) |
| Open | `http://<NAS-IP>:8051/login` (container port 8080) |
| Runs as | non-root, `user: "1000:1000"` |
| Persistent data | `/var/jenkins_home` → `/Volume1/DockerAppData/jenkinsdocker/var_jenkins_home` |
| Logs | `docker logs jenkinsdocker` (stdout/stderr) |
| Health check | `curl -fsS http://127.0.0.1:8080/login` every 30 s |
| Usage statistics / telemetry | disabled with `-Dhudson.model.UsageStatistics.disabled=true` |

### Why the app opens `/login`

Jenkins returns HTTP 403 on `/` for an anonymous visitor before login (it then
redirects the browser to the login page). The App Center opens the URL in
`config.ini`, so the package points it at `/login`, which returns HTTP 200 and
shows the login form. The health check probes the same page. This does not
change the port or weaken authentication.

## Privacy

The full policy is [`PRIVACY.md`](PRIVACY.md); it is also linked from the
App Center **help** link. In short:

- Source code, build logs, artifacts, credentials, user accounts and
  configuration stay in the application data directory on your NAS. The packager
  receives none of it.
- Jenkins **usage statistics and telemetry are disabled** by default in this
  package. The remaining automatic requests are the Jenkins update-centre check,
  its Google connectivity check, and Docker Hub when the image is pulled.
- Your own jobs and plugins contact the servers you configure (Git hosts, package
  registries, deployment targets, chat/e-mail, identity providers). Those flows
  are described in `PRIVACY.md` §4; restrict egress to what your builds need.
- Credentials are encrypted by Jenkins, but the key is stored in the same data
  directory. Protect the directory and every backup as confidential.

## Notices and licence

- The packaging is MIT — see [`LICENSE`](LICENSE).
- Upstream Jenkins and the official Docker image are MIT. The image
  `jenkins/jenkins:2.580.2-lts-jdk21` has multi-arch index digest
  `sha256:74401de6a6f92e1afe1b02d1c8e3d0c33f263bfb5275dd8813f9150355f2367b`.
- This is an independent community package by Moechz, not an official Jenkins
  distribution and not affiliated with the Jenkins project or TerraMaster.
- See [`NOTICE.md`](NOTICE.md) for the full third-party notice.

## Repository layout

```text
.
├── src/
│   ├── config.ini             # TOS metadata (id, name, path, help, version, platform)
│   ├── docker-compose.yml     # single Jenkins service
│   ├── jenkinsdocker.lang     # 23-language UI strings
│   └── jenkinsdocker.svg      # icon
├── scripts/
│   ├── build.sh               # build one architecture
│   └── build-dual.sh          # build both + official -c check
├── tools/
│   └── verify.py              # packaging self-check (run by build.sh)
├── test/
│   └── install-and-verify.sh  # on-device E2E (needs root; not run by CI)
├── TOSAppSelfTestingTool      # official TerraMaster package checker
├── PRIVACY.md
├── NOTICE.md
├── CHANGELOG.md
└── tosapp.json                # build spec: image, version, ports, volumes, translations
```

## Data, backup and uninstall

All persistent data is in:

```text
/Volume1/DockerAppData/jenkinsdocker/var_jenkins_home/
```

- The directory survives restarts, image updates and App Center uninstall. It
  contains `config.xml`, `jobs/`, `workspace/`, `users/`, `secrets/`,
  `credentials.xml`, `plugins/`, `logs/`, `updates/` and the other Jenkins data
  described in `PRIVACY.md` §2.
- **Back up before an upgrade or uninstall.** Stop the container or use a
  Jenkins-aware backup; copy the whole `var_jenkins_home/` directory. The backup
  contains credentials and their encryption keys.
- **Delete:** uninstall the app or run `docker compose -p jenkinsdocker down`,
  then verify and remove `/Volume1/DockerAppData/jenkinsdocker`. Removing the
  container alone keeps the data. Then remove the image with
  `docker rmi jenkins/jenkins:2.580.2-lts-jdk21` if it is no longer used.
- The package uses a bind mount, not a named Docker volume, so there is no hidden
  volume to clean up. Docker's own stdout/stderr log is managed by the Docker log
  driver.

## Upgrading from 2.541.3-2

The image moves from Jenkins LTS **2.541.3 (JDK 17)** to LTS **2.580.2
(JDK 21)**. Jenkins data and job formats are compatible, but the runtime and
plugins need attention:

1. Back up `var_jenkins_home/` (or a Jenkins-aware backup) and note the current
   image tag for rollback.
2. Stop the container, update the image tag in `docker-compose.yml`, and start
   it again. The package already does this; for an existing manual install,
   replace `jenkins/jenkins:2.541.3-lts-jdk17` with
   `jenkins/jenkins:2.580.2-lts-jdk21`.
3. Jenkins runs its normal upgrade on start. Open **Manage Jenkins → Plugins**
   and update plugins; 2.580.2 fixes high-severity core issues from the
   2026-10-28 advisory (CVE-2026-117612, -117613, -117614, -117615, -117616,
   -117617).
4. Java 17 → 21: plugins compiled for Java 17 run on 21. A plugin that is not
   compatible with 2.580.2 will be disabled with a warning; update or remove it.
   If a critical plugin is not yet compatible, restore the backup and roll back
   to the previous image tag until the plugin is updated.
5. After the upgrade, verify the login page, a simple job, credentials and agent
   connections.

## Build and verification

```bash
# both architectures + official package check
scripts/build-dual.sh 2.580.2-1

# the build writes:
#   out/jenkinsdocker_x86_64.tar.gz  + .sha256
#   out/jenkinsdocker_aarch64.tar.gz + .sha256

# isolated runtime check (random loopback port, temporary data directory)
python3 ../stage.py . --timeout 420
```

`scripts/build-dual.sh` runs `tools/verify.py` (archive layout, JSON validity,
required fields, 23-language coverage, version consistency, Docker-Hub-only
image, fixed tag, reserved/recommended host port, non-root user, per-service
healthcheck/TZ/restart, data path below the application data root, no `/Volume*`
wildcard, no literal secrets, icon size/element limits) and the official
`TOSAppSelfTestingTool -c` on both packages.

> Do **not** run `tosdocker e2e` on a live NAS: it deletes
> `/Volume1/DockerAppData/<appid>`. Use `stage.py` for isolated verification.

## Store submission

Both architectures are submitted as separate App Center records with the same
`app_id`:

| Architecture | Asset |
|---|---|
| `x86_64` | `jenkinsdocker_x86_64.tar.gz` |
| `aarch64` | `jenkinsdocker_aarch64.tar.gz` |

Version: `2.580.2-1` (upstream `2.580.2`, packaging iteration 1). Submission
remark: “Rectified per review comments.”

## Limitations

- Plain HTTP on port 8051. Put Jenkins behind a TLS reverse proxy or VPN before
  exposing it outside the LAN.
- No built-in MFA; the default authorisation is “logged-in users can do
  anything”. Use a matrix authorisation strategy and strong accounts, or add an
  MFA plugin.
- No automatic backup; the data directory and its encryption keys are your
  responsibility.
- Jenkins plugins are installed by the administrator and can make their own
  network requests. Review plugin security advisories and privacy behaviour.
- The package does not include an external database, a reverse proxy, or a
  hosted service. Jenkins stores everything locally.

---

Packaged for TOS 7 by Moechz. Upstream image used unmodified from Docker Hub.
