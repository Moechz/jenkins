# Review remediation — App Center listing review 2026-11-21

- **App:** Official Jenkins Docker image (`jenkinsdocker`), TerraMaster TOS 7
- **Reviewed submission:** `2.541.3-2` (x86_64; aarch64 also `status=0`)
- **Remediated submission:** `2.580.2-1` (x86_64 + aarch64)
- **Review codes:** C3, C4, C6, C8, T1 — plus the App Center self-test
  “online installation failed — cannot be opened” reported in the same notice.
- **Submission remark:** “Rectified per review comments.”

## Mapping

| Code | Review finding | What changed | Where to verify |
|---|---|---|---|
| C3 | Privacy policy is missing | Added `PRIVACY.md` with the full policy; `config.ini` and `tosapp.json` `help` now point to the public URL. The policy is in the repository, reachable in the package via the help link. | `PRIVACY.md`, `src/config.ini`, `tosapp.json` |
| C4 | Privacy policy is incomplete (retention, user rights, security measures, third-party sharing, deletion channel) | The policy has dedicated sections for stored data and retention (§2, §5), user rights and how to exercise them (§6), security measures and limitations (§7), third-party endpoints (§4) and deletion/uninstall (§10), plus contact (§11). | `PRIVACY.md` |
| C6 | Data collection inconsistent with the declaration | Upstream Jenkins enables anonymous usage statistics by default; the old package did not declare it. The package now sets `-Dhudson.model.UsageStatistics.disabled=true` (disables both usage statistics and telemetry), documents the remaining update-centre/Google/Docker Hub traffic, and explains that job/plugin traffic is administrator-controlled. | `src/docker-compose.yml`, `PRIVACY.md` §3–§4, `README.md` |
| C8 | Third-party API usage not disclosed | `PRIVACY.md` §4 is an endpoint table: `updates.jenkins.io` / `mirrors.updates.jenkins.io` / `*.cloudflare.jenkins.io` / `get.jenkins.io`, `www.google.com`, `usage.jenkins.io` (disabled), `uplink.jenkins.io` (disabled), Docker Hub (`registry-1.docker.io`, `auth.docker.io`, CDN), `repo.jenkins-ci.org` (incrementals only), website links, and the administrator-configured job/plugin destinations. Each row lists what is sent and the destination/region. | `PRIVACY.md` §4 |
| T1 | High/critical CVE | Upgraded `jenkins/jenkins:2.541.3-lts-jdk17` → `jenkins/jenkins:2.580.2-lts-jdk21` (current LTS, multi-arch). 2.580.2 fixes the high-severity core issues in the 2026-10-28 advisory. | `src/docker-compose.yml`, `tosapp.json`, `CHANGELOG.md`, `NOTICE.md` |
| App Center self-test | Online installation failed — cannot be opened | `config.ini` `path` changed from `http://${ip}:8051` to `http://${ip}:8051/login`; Jenkins returns 403 on `/` before login but 200 on `/login`. Health check probes `/login` with `-f`. Host port unchanged (8051). | `src/config.ini`, `src/docker-compose.yml`, `README.md` |

## T1 detail and migration note

The review's vulnerability-scan PDF was not delivered with the mail, so the
remediation is based on the upstream security advisory and the image version:

- The old image is Jenkins LTS **2.541.3** (JDK 17). The 2026-10-28 Jenkins
  security advisory states that Jenkins 2.587 and earlier, **LTS 2.580.1 and
  earlier**, are affected by six high-severity issues:
  - CVE-2026-117612 — XStream deserialization RCE via `resolves-to`;
  - CVE-2026-117613 — collection element type check bypass (stored XSS and
    other impacts);
  - CVE-2026-117614 — agent launch command approval bypass;
  - CVE-2026-117615 — symlink following on item rename (arbitrary file read);
  - CVE-2026-117616 — agent-name path traversal;
  - CVE-2026-117617 — artifact extraction race (arbitrary file creation).
- The new image is LTS **2.580.2**, which contains the fixes
  (<https://www.jenkins.io/security/advisory/2026-10-28/>). It is the current
  LTS at the time of submission (`updates.jenkins.io/stable/latestCore.txt` →
  `2.580.2`).
- **Java 17 → 21:** 2.580.2 is tested with JDK 21/25; the JDK 17 image tag is
  no longer published for this line. Jenkins data (`JENKINS_HOME`) and job
  formats are compatible. Plugins compiled for Java 17 run on Java 21; a plugin
  that is incompatible with 2.580.2 is disabled with a warning.
- **Upgrade path:** back up `var_jenkins_home/` (contains credentials and their
  keys), stop the container, replace the image tag, start it, update plugins in
  **Manage Jenkins → Plugins**, then verify the login page, a simple job,
  credentials and agents. Roll back by restoring the backup and the old image
  tag if a critical plugin is not yet compatible. Documented in `README.md`
  (“Upgrading from 2.541.3-2”).
- No 2.541.3 bits remain in the compose file; the image tag and `tosapp.json`
  are pinned to 2.580.2-lts-jdk21.

## C6 detail — what was actually inconsistent

Upstream Jenkins has two opt-out mechanisms:

1. **Usage statistics** (`hudson.model.UsageStatistics`): enabled when
   `noUsageStatistics` is unset, i.e. **on by default**. When due, the browser of
   a logged-in user loads `https://usage.jenkins.io/usage-stats.js` with the
   report data.
2. **Trial telemetry** (`jenkins.telemetry.Telemetry`): server-side POSTs to
   `https://uplink.jenkins.io/events` while a trial is active.

The old package had no privacy policy and did not declare either. The new
package sets:

```yaml
environment:
  JAVA_OPTS: "-Dhudson.model.UsageStatistics.disabled=true"
```

This sets `hudson.model.UsageStatistics.DISABLED = true` and makes
`jenkins.telemetry.Telemetry.isDisabled()` return `true`. Verified on the
packaged image at startup:

```text
CHECK UsageStatistics.DISABLED=true
CHECK isUsageStatisticsCollected=true   # the UI flag; the static DISABLED wins
CHECK Telemetry.isDisabled=true
```

With the property present the Jenkins UI cannot re-enable usage statistics; it
shows the option as disabled by a system property. The policy also explains what
would be sent if an administrator deliberately removes the property.

## C8 detail — endpoints and data

| Endpoint | Trigger | Data beyond IP/headers | Destination |
|---|---|---|---|
| `updates.jenkins.io` → `mirrors.updates.jenkins.io` / `*.cloudflare.jenkins.io` | startup + about daily; Plugin Manager | Jenkins version, requested metadata path | Jenkins project CDN (EU/US edges) |
| `get.jenkins.io` | plugin/core download on request | plugin name/version or core version | Jenkins project CDN |
| `www.google.com` | update-centre connectivity check | plain HTTP request | Google (US/global) |
| `usage.jenkins.io` | **disabled**; would be monthly from the browser | version, plugin list, OS/JVM, executor/job counts, random install id | Jenkins project |
| `uplink.jenkins.io` | **disabled**; would be server-side trial events | trial-specific technical data (system properties, security config, etc.) | Jenkins project |
| Docker Hub (`registry-1.docker.io`, `auth.docker.io`, CDN) | image pull/update by Docker Engine | image name/tag, Docker client headers | Docker, Inc. (US/global) |
| `repo.jenkins-ci.org/incrementals` | only incremental/experimental plugin builds | artifact coordinates | Jenkins project |
| Job/plugin destinations | when a job runs or a plugin is configured | source, artifacts, logs, credentials, notifications, account data | wherever the administrator configures |

No packager-side endpoint exists: the package does not phone home, check a
licence, or upload crash reports.

## App Center self-test — root cause and fix

The package itself passes `TOSAppSelfTestingTool -c`, and `stage.py` reports the
container running/healthy with the correct non-root user. The concrete
packaging defect was the URL the App Center opens:

- Jenkins answers `GET /` with **HTTP 403** for an anonymous visitor and only
  then sends a meta-refresh to `/login` (measured on the image).
- The App Center opens `config.ini.path` and treats a page that cannot be opened
  as a self-test failure. Pointing it at `/` therefore fails even though the
  service is up.
- `GET /login` returns **HTTP 200** with the login form both before and after
  setup. The package now uses `http://${ip}:8051/login` and probes the same page
  in the health check.
- The host port is unchanged at **8051**. No port was moved, per the packaging
  rules.

## Verification for this submission

| Gate | Result |
|---|---|
| `scripts/build-dual.sh 2.580.2-1` (self-check + official `-c`, both arches) | see build log below |
| `python3 stage.py work/jenkins --timeout 420` | see staging log below |
| Runtime: `/` → 403, `/login` → 200, image 2.580.2, telemetry disabled | see probe below |
| Anonymous release asset reachability | checked by `ghops.py publish` |

### Build

```text
== build jenkinsdocker 2.580.2-1 (aarch64)
verify OK: files=['config.ini', 'docker-compose.yml', 'jenkinsdocker.lang', 'jenkinsdocker.svg'] langs=23 icon_elements=4 icon_bytes=4749 host_port=8051 volumes=1 mode=plain
packed out/jenkinsdocker.tar.gz 7969 bytes
99fd08aabd189afd54f66c879806c96e8fe63a3523ca7761c5b45e2899f8b2c2  out/jenkinsdocker_aarch64.tar.gz
== build jenkinsdocker 2.580.2-1 (x86_64, canonical)
verify OK: ... host_port=8051 volumes=1 mode=plain
packed out/jenkinsdocker.tar.gz 7970 bytes
cbe1195f170b595b49010c9180edf038f6b9d12b872fa6284269fb2e1a6650f9  out/jenkinsdocker_x86_64.tar.gz
== official check (both architectures)
check passed
check passed
```

### Staging

```text
== staging jenkinsdocker 2.580.2-1 (隔离端口 + /tmp 数据)
   port map: jenkinsdocker 8051 -> 127.0.0.1:39855 (container 8080)
  [ OK ] 1 个容器 running 且 health 正常
  [ OK ] jenkinsdocker: 127.0.0.1:39855 -> HTTP 403
  [ OK ] stagejenkinsdocker-jenkinsdocker-1: user=1000:1000 (非 root)
  [staging] passed 3, failed 0  (keep=False)
```

`stage.py` probes `/` and accepts any HTTP response, so its 403 line is
expected; the health check (which probes `/login`) is what proves the page is
usable. The kept-container probe below shows 403 on `/` and 200 on `/login`.

### Runtime probe

```text
GET /       -> HTTP 403
GET /login  -> HTTP 200
X-Jenkins: 2.580.2
X-Hudson: 1.395
Config.Image: jenkins/jenkins:2.580.2-lts-jdk21
Config.User: 1000:1000
State.Health.Status: healthy
JAVA_OPTS=-Dhudson.model.UsageStatistics.disabled=true

CHECK UsageStatistics.DISABLED=true
CHECK isUsageStatisticsCollected=true   # UI flag; the static DISABLED wins
CHECK Telemetry.isDisabled=true
```

The last three lines were produced by a temporary `init.groovy.d` script in a
throw-away `JENKINS_HOME` using the same image and environment; the script and
the throw-away home were removed afterwards.

## Notes and residual risks

- The review's vulnerability-scan PDF is not available locally; the T1
  remediation is verified against the upstream advisory and the 2.580.2 image.
  If the reviewer's re-scan still reports a component, the PDF is needed to
  identify it precisely.
- The `aarch64` package is built and the image index contains `linux/arm64`, but
  the local staging run is on x86_64; arm64 was not booted on real hardware in
  this environment.
- The package deliberately keeps host port 8051 (no port change). If the
  reviewer's NAS already uses 8051 for another app, the install will fail on the
  port bind; that is a host conflict, not a package defect.
- Usage statistics and telemetry are disabled, but the update-centre and Google
  connectivity requests remain. They are disclosed; blocking them makes Jenkins
  work offline with update/plugin-install warnings.
