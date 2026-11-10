# Changelog

All notable user-visible changes to the Official Jenkins Docker image TOS application package.

## 2.541.3-1 — 2026-11-11

### Added
- First release. Official Jenkins Docker image (`jenkins/jenkins:2.541.3-lts-jdk17`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8051 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/jenkinsdocker/`.
