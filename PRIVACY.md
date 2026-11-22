# Privacy policy — Jenkins for TerraMaster TOS 7

**Applies to:** the TOS application package `jenkinsdocker` published by Moechz
(version `2.580.2-1` and later), which runs the upstream container image
`jenkins/jenkins:2.580.2-lts-jdk21` from Docker Hub unmodified.

**Operator of this package:** Moechz (see [Contact](#11-contact-and-changes)).
Jenkins itself is developed by the Jenkins project
(<https://www.jenkins.io/>), which is not the operator of this package.

**Summary:** Jenkins is a self-hosted CI/CD server. Source code, build logs,
artifacts, credentials, user accounts and configuration stay in the application
data directory on your own NAS. The packager does not receive, host or have
access to any of it. This package **disables Jenkins usage statistics and
telemetry by default**; the remaining automatic outbound traffic is the Jenkins
update-centre check, its connectivity check, and Docker Hub when the image is
pulled. The important exception is your own CI work: jobs and plugins contact
the servers you configure (Git repositories, package registries, deployment
targets, chat/e-mail services, and so on). Those transfers are described in
[section 4](#4-third-party-endpoints-and-what-is-sent).

Last updated: 2026-11-22.

---

## 1. Who is responsible

The application runs entirely on your hardware and under your control. The
packager (Moechz) does not operate a service that receives application data,
does not provide an account for this app, and cannot read your Jenkins
instance. You, as the NAS administrator, are the controller of the data in the
Jenkins instance. The packager is not a processor for that data.

The only data the packager may receive is what you choose to send in a public
GitHub issue (for example a bug report). Do not put credentials, build logs or
other secrets in an issue.

## 2. What is stored, and where

All persistent application data lives below:

```text
/Volume1/DockerAppData/jenkinsdocker/var_jenkins_home/
```

That directory is mounted as `/var/jenkins_home` inside the container
(`JENKINS_HOME`). The container runs as uid/gid `1000:1000` and has no Docker
socket, no host network and no privileged mode. Nothing else on the NAS is
mounted by this package.

| Path below `var_jenkins_home/` | What it contains | Retention |
|---|---|---|
| `config.xml` | Jenkins system configuration: system message, security realm and authorisation strategy, agent port, clouds, tool installations, proxy/SMTP and plugin settings. Some plugins store their configuration here, and a few plugin settings may be in clear text. | Until you change it or delete the data directory. |
| `credentials.xml`, `secrets/` | Credential definitions and their encrypted values; `secrets/master.key`, `secrets/hudson.util.Secret`, `secret.key`; the first-run `secrets/initialAdminPassword`; the CSRF/crumb salt and other instance keys. | Until you delete the credential or the data directory. The initial password file is no longer needed after setup but is not removed automatically. |
| `users/` | Jenkins user accounts when the built-in user database is used: user id, full name, optional e-mail address, bcrypt password hash, API-token entries, public keys/SSH keys, user preferences and a per-user seed. If you authenticate through LDAP, Active Directory, OIDC or SAML, the credentials are handled by that provider and Jenkins stores the resulting account/session data. | Until you delete the user or the data directory. |
| `jobs/` | Job and Pipeline definitions (`config.xml`), build records (`builds/<n>/build.xml`), build logs (`builds/<n>/log`), archived artifacts (`builds/<n>/archive/`), changelogs, fingerprints and job-level plugin data. | Jenkins does **not** discard old builds by default. Configure “Discard old builds” per job, or delete builds manually. |
| `workspace/` | The checked-out source tree and working files of each job (`workspace/<job>` by default). This is often the largest and most sensitive directory: it can contain source code, `.env` files, test data and build output. | Until the next build overwrites it, you run a workspace cleanup, or you delete the job/data directory. |
| `plugins/` | Installed plugin binaries (`.jpi`/`.hpi`), plugin dependencies and plugin-owned data. | Until you uninstall the plugin or delete the data directory. |
| `logs/` | Jenkins system and task logs (`java.util.logging` files), in addition to the container's stdout/stderr which Docker stores outside the app data directory. | Rotated according to Jenkins' logging configuration; the Docker log driver keeps its own copy. |
| `updates/` | Cached update-centre metadata (`default.json`) and downloaded update data. | Refreshed about daily; safe to delete while Jenkins is stopped. |
| `jenkins.telemetry.Correlator.xml` | A random correlation id generated at first start. It is only attached to telemetry if telemetry is enabled (it is disabled in this package). | Until you delete the data directory. |
| `war/` | Exploded Jenkins web application cache, regenerated from the image. | Until the data directory is deleted or the cache is cleared. |
| `userContent/` | Files you place there to be served by Jenkins, plus the shipped `readme.txt`. | Until you delete them. |
| `nodes/`, `fingerprints/`, `caches/`, `tools/` | Agent definitions, artifact fingerprints, caches and tool installations created by Jenkins or its plugins, when those features are used. | Feature-dependent; until deleted. |
| `copy_reference_file.log` | List of files copied from the image into `JENKINS_HOME` on start. | Until you delete it. |

Inside the container, temporary files may be written to `/tmp` (uploads,
workspace staging, plugin temp files). `/tmp` is part of the container's
writable layer and is discarded when the container is recreated; it is not part
of the NAS data directory.

### Credentials are encrypted, but the key is on the same disk

Jenkins encrypts the values stored in `credentials.xml` with AES-128 through
`hudson.util.Secret`. The encryption key is kept in the same `JENKINS_HOME`
(`secrets/hudson.util.Secret`, protected by `secrets/master.key`; legacy
`secret.key` files may also be present). This protects against casual reading of
a single file, but **not** against someone who can read the whole data directory
or a full backup: with the key and the encrypted values together, the
credentials can be decrypted. The same applies to passwords, API tokens and
other secrets that plugins store in their own configuration files.

Treat the whole `var_jenkins_home/` directory and every backup of it as
confidential. Restrict access to the NAS share, encrypt backups, and do not
publish the data directory.

## 3. What does *not* happen

- The package does not contact the packager. There is no licence check, no
  activation, no crash upload to Moechz, and no advertising or analytics SDK.
- The package does not send your jobs, source code, build logs, artifacts,
  credentials, user names or e-mail addresses to the packager.
- Jenkins **usage statistics are disabled** in this package by the Java system
  property `-Dhudson.model.UsageStatistics.disabled=true`. The Jenkins UI shows
  the Usage Statistics option as disabled by a system property.
- Jenkins **telemetry trials are disabled** by the same property. The server
  does not POST trial events to `uplink.jenkins.io`.
- The package does not include a reverse proxy, a CDN account, a hosted
  database, or any cloud component. Data does not leave the NAS unless a
  feature described in section 4 is used.

If you remove the `JAVA_OPTS` entry from the compose file (or set the property
to `false`), Jenkins may resume its upstream usage-statistics and telemetry
behaviour. That is your choice as administrator; section 4 describes what those
mechanisms send.

## 4. Third-party endpoints and what is sent

Every HTTP request reveals the source IP address and standard HTTP headers
(such as `User-Agent`) to the destination. The table lists the additional
application data beyond that. “Disabled” means this package has switched the
feature off; “on request” means it only happens when an administrator uses that
feature.

| Endpoint / host | When it is contacted | Data sent beyond IP and standard headers | Destination / region | How to control it |
|---|---|---|---|---|
| `updates.jenkins.io` (redirects to `mirrors.updates.jenkins.io` and `*.cloudflare.jenkins.io`) | At startup and about once a day for the update-centre metadata; also when the Plugin Manager is opened. | Jenkins version and the requested update-centre/metadata path. No job, user or credential data. | Jenkins project update service, served through a Cloudflare CDN (EU/US edges; global anycast). | Use an internal update site/mirror, block the host at the firewall, or leave Jenkins offline. Offline mode only disables update/plugin installation; the rest of Jenkins works. |
| `get.jenkins.io` (also `updates.jenkins.io/download/...`) | Only when you install or update a plugin or download a Jenkins core update. | Requested plugin name/version or core version; Jenkins version headers. No job data. | Jenkins project download service (global CDN). | Install plugins from a local/offline mirror, or do not install plugins. |
| `www.google.com` | At startup / update-centre check, as the “connection check” URL shipped in the update-centre metadata. | A simple HTTP request; no application data beyond IP and headers. | Google (US, global). | Block it at the firewall. Jenkins may show an “offline” warning and plugin installation may fail, but builds still run. A custom update site can supply a different `connectionCheckUrl`. |
| `usage.jenkins.io` (`/usage-stats.js`) | **Disabled in this package.** If enabled, a logged-in user's browser loads this script when the footer is rendered and the monthly report is due. | If enabled: Jenkins version, plugin short names and versions, OS/JVM vendor/name/version, executor and node counts, counts of jobs by type, a random installation id, and servlet-container information. No job names, logs, source, credentials or user names. The report is encrypted with the Jenkins project's public key. | Jenkins project usage-statistics service (US/global). | Keep `-Dhudson.model.UsageStatistics.disabled=true` (default here). The UI cannot override a system-property disable. |
| `uplink.jenkins.io` (`/events`) | **Disabled in this package.** If enabled, the Jenkins server posts trial events while a telemetry trial is active. | If enabled: trial-specific technical data. Trials shipped in recent Jenkins cover Java system properties, security configuration, Content-Security-Policy settings, distributed builds, uptime, user languages, optional permissions and Stapler dispatch traces. System properties can include host paths and JVM options; treat this as technical configuration data. No job content by design. | Jenkins project telemetry service (US/global). | Keep `-Dhudson.model.UsageStatistics.disabled=true` (default here). |
| `registry-1.docker.io`, `auth.docker.io`, Docker Hub CDN (`production.cloudflare.docker.com` and similar) | When Docker Engine pulls or updates the `jenkins/jenkins:2.580.2-lts-jdk21` image (install, recreate, `docker pull`). Jenkins itself does not call Docker Hub. | Image name/tag, Docker client headers, registry token request. No Jenkins data. | Docker, Inc. (US; registry and CDN endpoints worldwide). | Use a registry mirror (as configured on many TOS systems), pull once and keep the image locally, or pre-load the image. |
| `repo.jenkins-ci.org` (`/incrementals`) | Only if you install an incremental plugin build or enable the experimental update centre. Not used by the default configuration. | Requested incremental artifact coordinates. | Jenkins project artifact repository (US/EU). | Do not enable incremental/experimental updates. |
| `plugins.jenkins.io`, `www.jenkins.io`, `stats.jenkins.io` | Only when a user clicks a documentation, plugin or statistics link in the UI. | Normal browser request for the page; no app data. | Jenkins project websites (global CDN). | Nothing is sent automatically; links can be blocked at the browser/proxy. |
| Your own job and plugin destinations — for example GitHub/GitLab/Bitbucket, Maven Central, npm/PyPI, container registries, S3/Azure/GCS, Kubernetes, SSH hosts, Slack/Teams/Discord/webhooks, SMTP servers, LDAP/AD/OIDC/SAML providers, artifact repositories and cloud APIs | When a job runs, when a plugin is configured to do so, or when a user signs in through an external identity provider. | Whatever the job or plugin is configured to send: source code, commit metadata, dependency requests, build artifacts, logs, environment variables, credentials, notification text, account attributes. This is the main data flow of a CI server and is entirely under your control. | Wherever you configure it; the provider's region applies. | Review every job and plugin; use least-privilege credentials; restrict egress with a firewall or proxy; disable unused plugins; keep secrets in Jenkins credentials rather than job scripts. |

### Note on `up.jenkins.io`

Some review templates mention `up.jenkins.io`. That hostname does not resolve
and is **not** contacted by Jenkins 2.580.2. The update centre is
`updates.jenkins.io` (and its mirrors/CDN), and trial telemetry is
`uplink.jenkins.io`. Both are listed in the table above.

### Jenkins update centre and the Google connectivity check

The default update site is `https://updates.jenkins.io/update-center.json`, which
redirects to a mirror/CDN. Jenkins uses it to learn about Jenkins core and plugin
updates. The metadata also supplies the connectivity-check URL
(`https://www.google.com/`). These requests do not contain job or user data, but
they do reveal that this NAS runs Jenkins, its version, and its IP address to the
update service and to Google. If that is not acceptable, use an internal update
site or block the hosts; Jenkins remains usable offline (plugin installation and
update notifications will not work).

### Jenkins usage statistics and telemetry (disabled here)

Upstream Jenkins enables anonymous usage statistics by default: when a
logged-in user opens a page whose footer is due for a report, the browser loads
`https://usage.jenkins.io/usage-stats.js` with the report data. Upstream
telemetry trials are sent by the server to `https://uplink.jenkins.io/events`
when a trial is active. Both mechanisms are disabled in this package with the
standard Jenkins system property:

```yaml
environment:
  JAVA_OPTS: "-Dhudson.model.UsageStatistics.disabled=true"
```

The property sets `hudson.model.UsageStatistics.DISABLED` to `true`, which also
makes `jenkins.telemetry.Telemetry.isDisabled()` return `true`; the Jenkins UI
cannot re-enable either feature while the property is present. This was verified
on the packaged image by reading both values at startup.

### What your own builds send

Jenkins is a CI server: its purpose is to fetch source, resolve dependencies,
run tests, publish artifacts and deploy. Those actions contact the systems you
configure. They can transfer source code, credentials, build logs, artifacts and
environment variables to those systems. The packager has no visibility into and
no control over that traffic. Before exposing Jenkins to the internet, review
every job, every plugin and every credential, and restrict outbound traffic to
the destinations your builds actually need.

## 5. Retention periods

There is no packager-side retention because the packager receives no data. On
the NAS:

| Data | Default retention | How to shorten it |
|---|---|---|
| Jobs, builds, build logs, artifacts | Kept indefinitely; Jenkins does not discard old builds unless you configure it. | Enable “Discard old builds” in each job, or delete builds. |
| Workspaces | Kept until overwritten by the next build or cleaned. | Add a workspace cleanup step; delete `workspace/<job>`. |
| User accounts, credentials, configuration | Kept until you delete them. | Delete the user/credential in the UI, or delete the data directory. |
| System/task logs | Rotated by Jenkins' logging configuration; Docker's stdout/stderr log is retained by the Docker log driver. | Configure Docker log rotation; delete `logs/` while stopped. |
| Update-centre cache (`updates/`) | Refreshed about daily. | Delete it while Jenkins is stopped. |
| Telemetry correlation id | Kept for the life of the data directory; not transmitted while telemetry is disabled. | Delete the data directory. |
| Backups and snapshots | Until you delete them. They contain the encryption keys and all Jenkins data. | Apply your backup retention policy and securely erase expired copies. |

Deleting a job does not necessarily delete its workspace immediately; deleting
the data directory is the complete removal.

## 6. Your rights and how to exercise them

Because the packager does not hold your data, you exercise these rights directly
on the NAS and in Jenkins. The table is a practical map.

| Right | How to exercise it |
|---|---|
| Access / copy (portability) | Read the data directory, use Jenkins' UI/CLI/REST API, or copy `var_jenkins_home/` while Jenkins is stopped. Job configuration is XML; build records and logs are files. |
| Correction | Edit users, credentials, jobs, plugins and system configuration in the Jenkins UI or in the files while stopped. |
| Erasure | Delete individual jobs/builds/users/credentials in Jenkins, then remove the data directory after uninstalling. See [section 10](#10-deletion-and-uninstall). |
| Restrict processing / object | Stop the container, block outbound traffic at the firewall, disable or uninstall plugins, and do not configure external services. |
| Withdraw consent | No consent is collected by the packager. For optional features (usage statistics, telemetry, external plugins), remove the feature or the corresponding configuration. |
| Complaint | Contact the packager through <https://github.com/Moechz/jenkins/issues>; you also have the right to contact your local data-protection authority. |

There is no packager account to delete and no packager-side profile. If you
posted data in a public GitHub issue and want it removed, ask in that issue; note
that GitHub's own retention applies to issue history.

## 7. Security measures

What the package does:

- Runs the container as the non-root user `1000:1000`; no `privileged`, no
  `network_mode: host`, no `cap_add`, no Docker socket and no host directories
  other than the app data directory.
- Publishes only the web port (host 8051 → container 8080). The Jenkins agent
  port 50000 is exposed inside the container image but is not published by this
  package.
- Pins the image to the fixed tag `jenkins/jenkins:2.580.2-lts-jdk21` (never
  `:latest`) from Docker Hub, with a recorded digest.
- Keeps all persistent data below the application data directory.
- Disables upstream usage statistics and telemetry by default.
- Ships a health check that probes the login page, so a failed Jenkins start is
  visible to the platform.

What you must do:

- **Use TLS for remote access.** The package serves plain HTTP on port 8051.
  Put it behind a TLS reverse proxy or a VPN before exposing it outside the LAN.
- **Complete the setup wizard and change the initial admin password.** The
  first-run password is in `secrets/initialAdminPassword`; delete that file
  after setup. Keep “Logged-in users can do anything” only if you trust every
  account; otherwise use a matrix-based authorisation strategy.
- **Protect the data directory and backups.** Credentials are encrypted with a
  key stored in the same directory; the encryption is not a substitute for
  filesystem permissions. Restrict the NAS share, encrypt backups, and remember
  that build logs and workspaces often contain secrets in clear text.
- **Keep Jenkins and plugins updated.** Jenkins core and plugins have a history
  of security advisories, including the high-severity issues fixed in 2.580.2.
  Review plugin advisories and remove unused plugins.
- **Use least privilege.** Give jobs and agent/SSH credentials only the access
  they need; prefer short-lived tokens; do not run builds as a NAS administrator.
- **Restrict egress.** Firewall the container to the destinations your builds
  and update process actually need. If you do not use update-centre features,
  block `updates.jenkins.io`, `get.jenkins.io` and `www.google.com` as well.
- **Control logs.** Build logs, Pipeline output and plugin logs can contain
  secrets; restrict access to `jobs/*/builds/*/log` and Docker logs, and rotate
  them.

Known limitations: no built-in TLS, no built-in MFA (plugins can add it), no
automatic backup, and no malware scanning of workspaces/artifacts. These are
normal for a self-hosted CI server but must be covered by your own controls.

## 8. Third-party services' own policies

The destinations in section 4 have their own privacy policies. Relevant links:

- Jenkins usage-statistics advisory: <https://www.jenkins.io/blog/2016/03/30/usage-statistics-privacy-advisory/>
- Jenkins security: <https://www.jenkins.io/security/>
- Docker: <https://www.docker.com/legal/privacy/>
- Google: <https://policies.google.com/privacy>
- Your SCM, registry, cloud, chat and identity providers: see the provider's
  policy.

The packager does not control those services and does not receive their data.

## 9. International transfers

The packager does not transfer data internationally because it receives none.
Automatic Jenkins requests may reach CDN/cloud endpoints outside your country
(typically EU/US edges for the Jenkins update service and Google, and Docker
Hub's global CDN). Your own jobs and plugins may transfer data to whatever
region you configure. If you need to keep all traffic in a region, use internal
mirrors, a local proxy, and region-local job destinations.

## 10. Deletion and uninstall

Removing the container does **not** remove the data directory. To delete
everything:

1. Back up anything you need first; deletion is irreversible.
2. Stop and remove the container:

   ```bash
   docker compose -p jenkinsdocker down
   ```

   (or uninstall the app in App Center).
3. Verify the path and delete the application data directory:

   ```bash
   ls -la /Volume1/DockerAppData/jenkinsdocker
   rm -rf /Volume1/DockerAppData/jenkinsdocker
   ```

4. Remove the image if it is no longer needed:

   ```bash
   docker rmi jenkins/jenkins:2.580.2-lts-jdk21
   ```

5. Delete backups, exports and any Docker log copies you made. There is no
   hidden copy outside these locations; the package uses a bind mount, not a
   named Docker volume.

If you only want to remove one job, build, user or credential, do that in the
Jenkins UI and then clean the corresponding workspace/backup copies.

## 11. Contact and changes

- **Contact / data requests:** <https://github.com/Moechz/jenkins/issues>
  (public issue tracker; do not include credentials, tokens, build logs or other
  secrets).
- **Policy URL:** <https://github.com/Moechz/jenkins/blob/main/PRIVACY.md>
- **Source / packaging:** <https://github.com/Moechz/jenkins>
- **Upstream:** <https://github.com/jenkinsci/docker> and
  <https://www.jenkins.io/>

This policy applies to version `2.580.2-1` and later packaging iterations of the
same upstream line. Material changes will be described in `CHANGELOG.md` and in
the release notes; the current version of this document is always at the policy
URL above.

---

**Scope note:** this policy describes the TOS packaging and the behaviour of the
unmodified upstream image. It does not replace the privacy policy of Jenkins,
Docker, your SCM/cloud providers, or any plugin you install. If you install a
plugin that contacts an external service, that plugin's documentation and
privacy policy apply to the data it sends.
