
# Introduction

WebGoat is a deliberately insecure web application maintained by [OWASP](http://www.owasp.org/) designed to teach web
application security lessons.

This program is a demonstration of common server-side application flaws. The
exercises are intended to be used by people to learn about application security and
penetration testing techniques.

**WARNING 1:** *While running this program your machine will be extremely
vulnerable to attack. You should disconnect from the Internet while using
this program.*  WebGoat's default configuration binds to localhost to minimize
the exposure.

**WARNING 2:** *This program is for educational purposes only. If you attempt
these techniques without authorization, you are very likely to get caught. If
you are caught engaging in unauthorized hacking, most companies will fire you.
Claiming that you were doing security research will not work as that is the
first thing that all hackers claim.*

![WebGoat](docs/images/webgoat.png)

# WebGoat — App Repo (DevSecOps CI Pipeline)

This repository holds the application source code (WebGoat — OWASP's deliberately vulnerable Java/Spring Boot application, used as the demo target) and the **Shift-Left Security CI pipeline** that runs every commit through build, static and composition security analysis, builds a Docker image, scans it, and on success automatically updates the image tag in the GitOps (Helm) repository.

Second of the three repositories in the project:

| Repo | Role |
|---|---|
| [webgoat-infra](https://github.com/lukaradenkovic2003/webgoat-infra) | AWS infrastructure (Terraform) |
| **WebGoat** (this repo) | Application code + DevSecOps CI pipeline |
| [webgoat-helm](https://github.com/lukaradenkovic2003/webgoat-helm) | Helm chart + ArgoCD GitOps + DAST |

## Docker image

The application is packaged on top of the `eclipse-temurin:25-jdk-noble` base image, runs as a non-root user (`webgoat`), exposes ports **8080** (WebGoat) and **9090** (WebWolf), and has a `HEALTHCHECK` defined against `http://localhost:8080/WebGoat/actuator/health`.

## CI/CD Pipeline (`.github/workflows/...`) — "App DevSecOps Pipeline"

Triggers on every push and pull request to `main`. Made up of 5 sequential jobs (each depends on the previous one via `needs:`):

### 1. `build-and-quality` — Build & Quality Check
- Checkout, set up JDK 25 (Temurin), Maven cache
- `./mvnw clean package -DskipTests` — build
- `./mvnw checkstyle:check` — runs but **does not block the pipeline** (`continue-on-error: true`)
- Uploads the build artifact (`webgoat-*.jar`) for use by later jobs

### 2. `sast` — SAST (SonarCloud)
- Static code analysis via **SonarCloud** (`sonar-maven-plugin`)
- Uses `SONAR_TOKEN`, `SONAR_PROJECT_KEY`, `SONAR_ORGANIZATION`
- The Sonar Quality Gate result is visible on the SonarCloud dashboard; if you want the build to explicitly fail on a failed Quality Gate, add `sonar.qualitygate.wait=true` to the analysis step

### 3. `sca-scan` — SCA (Software Composition Analysis)
- Resolves Maven dependencies (`mvnw dependency:resolve`)
- Runs a **Trivy filesystem scan** (`aquasecurity/trivy-action`) against `pom.xml` / the resolved dependency tree — this is classic SCA: it doesn't look at your code, it looks at the third-party libraries your code pulls in
- `severity: HIGH,CRITICAL` + `exit-code: 1` → **pipeline fails** if a HIGH/CRITICAL vulnerability is found in dependencies

#### Real finding: XStream RCE (CVE-2013-7285)

During development, Trivy's fs scan against `pom.xml` found **23 vulnerabilities**, all traced back to a single dependency: **XStream 1.4.5** (used for XML serialization/deserialization).

- **22 HIGH** + **1 CRITICAL**
- The critical one — **CVE-2013-7285** — allows **Remote Code Execution** through insecure XML deserialization. It's an old, well-documented vulnerability.

Because `sca-scan` has `severity: HIGH,CRITICAL` + `exit-code: 1`, Trivy's exit code 1 marked the job as **FAILED**, which meant `docker-build-and-scan` (which has `needs: sca-scan`) never ran — the Docker image was never built, let alone pushed to ECR. This is exactly the behavior the task requires: *"if SAST or SCA detect HIGH/CRITICAL vulnerabilities, the pipeline is halted"* and *"code with a critical vulnerability must not reach ECR."*

**Fix:** the vulnerable version was pinned via a property in `pom.xml`:

```xml
<xstream.version>1.4.5</xstream.version>
```

Bumping it to a patched release resolved the CVE (since `${xstream.version}` is referenced in two places in the file, the single property change updated both):

```xml
<xstream.version>1.4.20</xstream.version>
```

After the fix, the same scan with the same rules re-ran against the same `pom.xml`, found 0 (or far fewer) HIGH/CRITICAL results, returned exit code 0, and the pipeline proceeded to `docker-build-and-scan`. This before/after pair — a real CRITICAL finding that blocked the build, and a real fix that let it through — is good evidence that the Quality Gate works correctly in both directions: it blocks vulnerable code and doesn't needlessly block clean code, which is exactly what a shift-left security pipeline is for.

### 4. `docker-build-and-scan` — Docker Build & Image Scan
- Downloads the jar artifact from the first job, builds the Docker image (`docker build -t webgoat:<sha>`)
- Runs a **Trivy image scan** against the built image, again `HIGH,CRITICAL` + `exit-code: 1` → pipeline fails on critical vulnerabilities in the image itself
- AWS authentication goes through **GitHub OIDC** (no static AWS keys) — assumes the `github-actions-ecr-push` IAM role
- Logs in to ECR and pushes the image tagged with the commit SHA (`${{ github.sha }}`)

### 5. `update-helm-repo` — GitOps Trigger
- Checks out the `webgoat-helm` repo using `HELM_REPO_TOKEN` (a PAT with write access)
- Uses `sed` to update `tag:` in `values.yaml` to the new commit SHA
- Commits and **pushes directly** to `webgoat-helm` (note: this is currently a direct push to the helm repo's `main` branch, not a Pull Request — the task allows either approach; if you want a stricter flow, this step can easily be changed to open a PR instead)

After this step, **ArgoCD** (which watches the `webgoat-helm` repo) automatically detects the tag change and syncs the new version to the EKS cluster — closing the full GitOps loop from an application commit to a live deployment.

## Security Gates — what stops the pipeline

| Check | Tool | Behavior |
|---|---|---|
| Code quality | Checkstyle | Non-blocking (informational) |
| SAST | SonarCloud | Result visible on the dashboard |
| SCA (dependencies) | Trivy `fs` scan | **Fails the build** on HIGH/CRITICAL |
| Docker image | Trivy `image` scan | **Fails the build** on HIGH/CRITICAL |

> **Note:** the current pipeline has no explicit Slack notify step in the `WebGoat` repo (unlike `webgoat-infra`, which has one). If the team should get a Slack alert for failed SAST/SCA/Trivy checks too (not just see the build fail in the GitHub Actions UI), add a `curl` call to `SLACK_WEBHOOK_URL` in an `if: failure()` step for each relevant job, following the same pattern used in the `webgoat-infra` pipeline.

## GitHub Secrets & Variables

**Repository secrets:**
| Secret | Purpose |
|---|---|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | (legacy/fallback — the main flow uses the OIDC role) |
| `ECR_REGISTRY` / `ECR_REPO_URL` | ECR registry address for pushing images |
| `HELM_REPO_TOKEN` | PAT for pushing to the `webgoat-helm` repo (GitOps trigger) |
| `SONAR_TOKEN` | Authentication to SonarCloud |

**Repository variables:**
| Variable | Purpose |
|---|---|
| `AWS_DEFAULT_REGION` | `us-east-1` |
| `SONAR_ORGANIZATION` | SonarCloud organization |
| `SONAR_PROJECT_KEY` | SonarCloud project key |

## Running locally

```bash
./mvnw clean package -DskipTests
java -Dwebgoat.port=8080 -Dwebwolf.port=9090 -jar target/webgoat-*.jar
```

or via Docker:

```bash
docker build -t webgoat:local .
docker run -it -p 8080:8080 -p 9090:9090 webgoat:local
```

The application is available at `http://localhost:8080/WebGoat/login`.
