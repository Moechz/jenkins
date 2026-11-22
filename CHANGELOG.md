# Changelog

All notable user-visible changes to the Official Jenkins Docker image TOS application package.

## 2.580.2-1 — 2026-11-22

Remediation release for the App Center listing review of 2026-11-21
(`C3`, `C4`, `C6`, `C8`, `T1`, plus the App Center self-test “cannot be opened”
failure). Submission remark: “Rectified per review comments.”

### Security (T1)
- Upgraded the image from `jenkins/jenkins:2.541.3-lts-jdk17` to
  **`jenkins/jenkins:2.580.2-lts-jdk21`**, the current Jenkins LTS
  (2026-10-28), multi-arch `linux/amd64` + `linux/arm64`, from Docker Hub,
  unmodified.
- 2.580.2 fixes the high-severity Jenkins core issues in the 2026-10-28
  advisory: CVE-2026-117612 (XStream `resolves-to` RCE), CVE-2026-117613
  (collection element type check bypass), CVE-2026-117614 (agent launch command
  approval bypass), CVE-2026-117615 (symlink following on item rename),
  CVE-2026-117616 (agent-name path traversal) and CVE-2026-117617 (artifact
  extraction race). The previous 2.541.3 LTS is affected by all of them.
- Runtime moves from Java 17 to Java 21. Existing `JENKINS_HOME` data is
  compatible; see the upgrade notes below.

### Privacy and compliance (C3/C4/C6/C8)
- Added `PRIVACY.md`: what is stored (workspaces, builds/logs, artifacts,
  credentials, accounts, plugins), where it is stored, retention, user rights,
  security measures, deletion/uninstall, contact, and a full third-party
  endpoint table (update centre/CDN, Google connectivity check, Docker Hub,
  usage statistics, telemetry, job/plugin destinations).
- Added `NOTICE.md` with the upstream MIT licence, image tag and multi-arch
  index digest, trademark and affiliation notes.
- **Disabled Jenkins usage statistics and telemetry by default** with
  `-Dhudson.model.UsageStatistics.disabled=true`. Upstream Jenkins enables
  anonymous usage statistics by default, which was not declared in the old
  package; the new package turns both mechanisms off and documents the
  remaining update-centre/connectivity traffic.
- `help` now points to `https://github.com/Moechz/jenkins/blob/main/PRIVACY.md`
  in both `config.ini` and `tosapp.json`.
- README documents the repository layout, privacy, notices, build, data,
  backup, uninstall and the upgrade path.

### App Center self-test (cannot be opened)
- `config.ini` `path` now opens `http://${ip}:8051/login` instead of the root
  URL. Jenkins returns HTTP 403 on `/` before login and only then redirects the
  browser; `/login` returns HTTP 200 and shows the login form, so the App Center
  can open the application. The health check probes the same page with `-f`.
- The host port remains **8051**; no port change was made.

### Documentation
- `tosapp.json`, `CHANGELOG.md`, `README.md`, `LICENSE` third-party notice and
  all 23 language sections updated to the new image/version. Translated
  `zh-cn`/`zh-hk` descriptions and release notes rewritten for Jenkins (no
  text from other applications).
- Removed the stale `probe` block (old digest/trial data) from `tosapp.json`.

## 2.541.3-1 — 2026-11-11

### Added
- First release. Official Jenkins Docker image (`jenkins/jenkins:2.541.3-lts-jdk17`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8051 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/jenkinsdocker/`.
