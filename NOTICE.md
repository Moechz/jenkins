# Notices — Jenkins for TerraMaster TOS 7

This repository contains the TOS 7 **packaging** for the official Jenkins Docker
image. It does not contain Jenkins source code and does not modify the upstream
image.

## Upstream project and licence

| Item | Value |
|---|---|
| Upstream project | Jenkins — <https://www.jenkins.io/> |
| Upstream packaging repository | <https://github.com/jenkinsci/docker> |
| Container image | `jenkins/jenkins:2.580.2-lts-jdk21` |
| Image registry | Docker Hub (`docker.io/jenkins/jenkins`) |
| Multi-architecture index digest | `sha256:74401de6a6f92e1afe1b02d1c8e3d0c33f263bfb5275dd8813f9150355f2367b` |
| Architectures in the index | `linux/amd64`, `linux/arm64` (plus other platforms published by upstream) |
| Upstream licence | MIT (Jenkins core and the official Docker packaging) |
| Image licence label | `org.opencontainers.image.licenses = MIT` |

The image is pulled from Docker Hub at install time and is used **unmodified**.
No Jenkins source code or binaries are redistributed in this repository or in
the TOS package; the package contains only the four TOS metadata files
(`config.ini`, `jenkinsdocker.lang`, `jenkinsdocker.svg`,
`docker-compose.yml`). The image's own licence and notices apply to the image.

## This packaging

| Item | Value |
|---|---|
| Package ID | `jenkinsdocker` |
| Packager | Moechz |
| Packaging licence | MIT — see [`LICENSE`](LICENSE) |
| Package version | `2.580.2-1` |
| Upstream version | `2.580.2` (LTS) |

The MIT licence in [`LICENSE`](LICENSE) covers the packaging files authored for
this repository. It does not relicense Jenkins or any third-party component in
the container image.

## Trademarks and affiliation

Jenkins and the Jenkins logo are trademarks of the Jenkins project or their
respective owners. TerraMaster and TOS are trademarks of TerraMaster. Docker is
a trademark of Docker, Inc. All other trademarks are the property of their
respective owners.

This is an **independent community package** by Moechz. It is not an official
Jenkins distribution, is not endorsed by the Jenkins project, and is not
affiliated with TerraMaster beyond being submitted to the TerraMaster App
Center.

## TerraMaster tooling

`TOSAppSelfTestingTool` is an official TerraMaster utility, redistributed in
this repository unmodified for local package validation. It is not part of the
installed application and is not covered by the MIT licence above.

## Security and vulnerability remediation

The package uses the current Jenkins LTS line at the time of submission. The
2.580.2 release contains fixes for the high-severity Jenkins core issues
published in the 2026-10-28 security advisory, including CVE-2026-117612,
CVE-2026-117613, CVE-2026-117614, CVE-2026-117615, CVE-2026-117616 and
CVE-2026-117617. See <https://www.jenkins.io/security/advisory/2026-10-28/>.

Jenkins plugins are installed by the administrator after first start and are
not part of this package. Review plugin security advisories before installing or
updating them.
