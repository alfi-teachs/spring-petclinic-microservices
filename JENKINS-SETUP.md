# Jenkins CI Pipeline for Spring PetClinic Microservices (Local Lab)

Learning log of everything set up so far, step by step, with the "why" and interview notes.

**Repo:** `https://github.com/alfi-teachs/spring-petclinic-microservices` (fork of `spring-petclinic/spring-petclinic-microservices`)
**Machine:** Windows laptop, Git Bash, Docker Desktop (WSL2 engine)

---

## Architecture

```
GitHub (code + Jenkinsfile)
        │  (webhook – coming later via ngrok)
        ▼
┌─────────────── Docker Desktop · network "jenkins" ───────────────┐
│                                                                   │
│  jenkins-controller  ◄── WebSocket ──  jenkins-agent              │
│  (UI, jobs, creds,                     (Java 21, Maven wrapper,   │
│   0 executors)                          Docker CLI, Trivy)        │
│                                              │                    │
│                                              ├─► Docker Desktop engine (via docker.sock)
│                                              └─► SonarCloud (online)
└───────────────────────────────────────────────────────────────────┘
```

- **Controller** = teacher: hands out work, stores jobs and credentials, never builds.
- **Agent** = student: does all the work (compile, test, scan, build images).
- **SonarCloud** = examiner: checks code quality, returns PASS/FAIL.

---

## Step 1 – Clone the fork

```bash
cd /c/Users/almal/OneDrive/Desktop/lab/jen
git clone https://github.com/alfi-teachs/spring-petclinic-microservices.git
cd spring-petclinic-microservices
git remote -v
```

- `origin` = my fork (I push here, webhook goes here)
- `upstream` = original project (only pull updates from it)

> **Interview:** Fork = copy on GitHub. Clone = copy on my machine. `origin` is where I push; `upstream` is the original repo I fetch updates from.

## Step 2 – Understand the project

| Item | Meaning |
|---|---|
| 8 `spring-petclinic-*` folders | 8 microservices (config, discovery, gateway, customers, vets, visits, admin, genai) |
| `pom.xml` | Maven build file (multi-module) |
| `mvnw` | Maven Wrapper – no Maven install needed |
| `<java.version>17` | Project targets Java 17 |
| `docker/Dockerfile` | One shared Dockerfile for all services (uses `ARG ARTIFACT_NAME`) |

> **Interview:** Spring Boot microservices, Java 17, multi-module Maven with the wrapper, shared Dockerfile parameterised by artifact name.

## Step 3 – Docker Desktop checks and memory

```bash
docker version                                   # must show Client AND Server
docker info 2>/dev/null | grep -i "total memory"
```

- Docker Desktop runs on the **WSL2 engine** (`Kernel Version: ...microsoft-standard-WSL2`).
- WSL2 memory is set in `C:\Users\almal\.wslconfig` (not in Docker Desktop):
  ```ini
  [wsl2]
  memory=6GB
  swap=4GB
  ```
  Applies only after: Quit Docker Desktop → `wsl --shutdown` → start Docker Desktop.
- Memory still showed ~3.7 GB, so **Plan B**: SonarQube runs online (SonarCloud) instead of locally.

> **Interview:** Docker Desktop on Windows has two backends. WSL2 → resources in `.wslconfig`; Hyper-V → Docker Desktop Resources settings.

## Step 4 – Start the Jenkins controller (container)

```bash
docker network create jenkins
docker volume create jenkins_home

MSYS_NO_PATHCONV=1 docker run -d \
  --name jenkins-controller \
  --network jenkins \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  --restart unless-stopped \
  jenkins/jenkins:lts
```

| Part | Why |
|---|---|
| `--network jenkins` | Containers find each other by name (Docker DNS) |
| `-v jenkins_home:...` | Jobs/plugins/creds survive container restarts |
| `--restart unless-stopped` | Auto-start with Docker |
| `MSYS_NO_PATHCONV=1` | Stops Git Bash converting `/var/...` to a Windows path |

Containers are not tied to any folder – it doesn't matter where you run `docker run` from.

## Step 5 – Unlock Jenkins

```bash
MSYS_NO_PATHCONV=1 docker exec jenkins-controller cat /var/jenkins_home/secrets/initialAdminPassword
```

The file is **inside the container**, so `cat` on the laptop fails – use `docker exec`.
Open `http://localhost:8080` → paste password.

## Step 6 – Suggested plugins + admin user

- **Install suggested plugins** (Git, Pipeline, Credentials, GitHub, Email Extension, JUnit…)
- Create first admin user.
- Jenkins URL kept as `http://localhost:8080/` for now (the "use a valid host name" warning is fine; it becomes the ngrok URL later).

## Step 7 – Extra plugins

Manage Jenkins → Plugins → Available:

| Plugin | Why |
|---|---|
| Docker Pipeline | Build/push images from the Jenkinsfile |
| SonarQube Scanner | `withSonarQubeEnv`, `waitForQualityGate` |
| Pipeline: AWS Steps | `withAWS` (later) |
| AWS Credentials | Store AWS keys (later) |
| Pipeline Graph View | Stage boxes, click a red box to see its log |

Skipped: **Webhook Relay** (we'll use ngrok), **Blue Ocean** (no longer developed), **oxlint** (JavaScript linter – this is a Java project).

Restart: `http://localhost:8080/safeRestart` or `docker restart jenkins-controller`.

## Step 8 – Custom agent image

File `jenkins-agent/Dockerfile` (saved in the repo = infrastructure as code):

```dockerfile
FROM jenkins/inbound-agent:latest-jdk21
USER root
COPY --from=docker:cli /usr/local/bin/docker /usr/local/bin/docker
COPY --from=docker:cli /usr/local/libexec/docker/cli-plugins /usr/local/libexec/docker/cli-plugins
COPY --from=aquasec/trivy:latest /usr/local/bin/trivy /usr/local/bin/trivy
```

```bash
docker build -t petclinic-agent:1 jenkins-agent/
docker run --rm --entrypoint sh petclinic-agent:1 -c "java -version; docker --version; trivy --version"
```

- `FROM` = start from the official Jenkins agent (already knows how to connect to the controller).
- `USER root` = needed to copy tools and use the docker socket (lab shortcut; avoid in production).
- `COPY --from=` = multi-stage trick: take only the binary from another official image.
- Java 21 builds the Java 17 project fine (Spring Boot sets compiler `release=17`).

Two Dockerfiles – don't mix them up:
- `jenkins-agent/Dockerfile` → the **worker** that runs the pipeline
- `docker/Dockerfile` → the **petclinic app images** the pipeline builds

## Step 9 – Connect the agent (controller–agent architecture)

1. **Built-In Node → Configure → Number of executors = 0** (controller never builds).
2. **Manage Jenkins → Nodes → New Node** → `docker-agent`, Permanent Agent:
   - Executors: `1` · Remote root: `/home/jenkins/agent` · Label: `docker`
   - Usage: only jobs with matching label · Launch: *connecting it to the controller*
3. Copy **only the secret** (the code after `-secret`) from the node page – ignore the Windows/Unix commands.
4. Start the agent:

```bash
MSYS_NO_PATHCONV=1 docker run -d \
  --name jenkins-agent \
  --network jenkins \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e JENKINS_URL=http://jenkins-controller:8080 \
  -e JENKINS_AGENT_NAME=docker-agent \
  -e JENKINS_SECRET=<SECRET> \
  -e JENKINS_WEB_SOCKET=true \
  -e JENKINS_AGENT_WORKDIR=/home/jenkins/agent \
  petclinic-agent:1
```

Check:
```bash
docker logs jenkins-agent 2>&1 | tail -5        # INFO: Connected
docker exec jenkins-agent docker ps             # proves the docker.sock "cable" works
```
Check the label: `http://localhost:8080/label/docker/` (labels are case-sensitive).

> **Interview:**
> - **Controller–agent (master–slave):** controller schedules and stores config; agents (SSH or inbound/JNLP/WebSocket, picked by labels) run builds. Scales out, isolates builds.
> - **Node** = any machine that can run builds. **Executor** = a slot on a node running one build at a time. Controller set to 0 executors.
> - Label mismatch → build waits in the queue forever ("no nodes with label").
> - Mounting `docker.sock` = root on the host. Production: rootless Docker, Kaniko, or ephemeral Kubernetes agents.

## Step 10 – Credentials (all added on the controller)

Manage Jenkins → Credentials → System → Global credentials → Add Credentials.

| ID | Kind | What |
|---|---|---|
| `github-creds` | Username with password | GitHub username + classic PAT (scopes: `repo`, `admin:repo_hook`, 30-day expiry) |
| `dockerhub-creds` | Username with password | Docker Hub username + access token (Read & Write) |
| `sonar-token` | **Secret text** (change the Kind dropdown) | SonarCloud personal token |
| `aws-creds` | AWS Credentials | **Later** – IAM user `jenkins-ci`, `AmazonEC2ContainerRegistryPowerUser`, never root/admin keys |

SonarCloud setup:
- Logged in with GitHub, created a **free organization**, gave access to **only this repo** (no auto-import of all repos – least privilege).
- **Automatic Analysis OFF** (Jenkins does the scanning).
- Org key: `alfi-teachs` · Project key: `alfi-teachs_spring-petclinic-microservices`
- Manage Jenkins → System → **SonarQube servers**: name `sonarcloud`, URL `https://sonarcloud.io`, token `sonar-token`.

> **Interview:** Secrets live encrypted on the controller and are referenced by ID; agents get them only at runtime for the steps that ask, and they're masked in logs. Never hard-code secrets or pass them as command-line arguments.

## Step 11 – Jenkinsfile (Pipeline as Code), written piece by piece

`Jenkinsfile` = a **file** (not folder), exact name, project root next to `pom.xml`. Written in Groovy, **declarative** style (`pipeline { }`), not scripted (`node { }`).

Current version:

```groovy
pipeline {
    agent { label 'docker' }

    options {
        skipDefaultCheckout()
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                echo '>>> Step 1: Getting code from GitHub...'
                checkout scm
                echo '>>> Checkout finished ✅'
            }
        }

        stage('Compile') {
            steps {
                echo '>>> Step 2: Compiling Java code with Maven...'
                sh './mvnw -B clean compile'
                echo '>>> Compile finished ✅'
            }
        }

        stage('Test') {
            steps {
                echo '>>> Step 3: Running unit tests...'
                sh './mvnw -B test'
                echo '>>> Test finished ✅'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo '>>> Step 4: Sending code to SonarCloud for quality check...'
                withSonarQubeEnv('sonarcloud') {
                    sh '''./mvnw -B sonar:sonar \
                        -Dsonar.organization=alfi-teachs \
                        -Dsonar.projectKey=alfi-teachs_spring-petclinic-microservices'''
                }
                echo '>>> SonarQube analysis finished ✅'
            }
        }
    }
}
```

| Piece | Meaning |
|---|---|
| `agent { label 'docker' }` | Run on a node labelled `docker` |
| `skipDefaultCheckout()` | We check out in our own visible stage |
| `timestamps()` | Time on every log line |
| `buildDiscarder(... '10')` | Keep last 10 builds |
| `timeout(30 min)` | Kill hung builds |
| `checkout scm` | Clone the exact commit that triggered the build |
| `./mvnw -B clean compile` | `-B` = batch mode (clean CI logs) |
| `post { always { junit ... } }` | Publish test reports even if tests fail |
| `withSonarQubeEnv('sonarcloud')` | Injects Sonar URL + token secretly – don't add `sonar.host.url` / `sonar.login` yourself |
| `'''...'''` + `\` | Multi-line shell command (avoid `'...'\ + '...'` string joining – easy to break) |

Maven phases: `validate → compile → test → package → verify → install → deploy` (running one runs all before it).

## Step 12 – Pipeline job

New Item → `petclinic-pipeline` → **Pipeline**:
- Definition: **Pipeline script from SCM** · SCM: Git
- Repo URL: `https://github.com/alfi-teachs/spring-petclinic-microservices.git`
- Credentials: `github-creds` · Branch: `*/main` · Script Path: `Jenkinsfile`

Every change: **add → commit → push**, then Build Now:

```bash
git add Jenkinsfile
git commit -m "message"
git push origin main
git status        # must say "working tree clean"
```

## Results so far

| Build | Result |
|---|---|
| Checkout + Compile | ✅ SUCCESS |
| + Test (JUnit report published) | ✅ SUCCESS |
| + SonarQube | Analysis uploaded, **Quality Gate: Passed** on SonarCloud, but the **Jenkins build showed FAILURE – cause still being investigated** |

Note on the Quality Gate: the first scan has no "new code", so it passes easily. Coverage shows 0% because the project has no JaCoCo yet.

---

## Troubleshooting learned

| Symptom | Cause / fix |
|---|---|
| `docker info` memory unchanged | Docker VM not restarted: Quit Docker Desktop → `wsl --shutdown` → start again |
| `cat /var/jenkins_home/...: No such file` | File is inside the container → `docker exec` |
| Build stuck in queue | Label mismatch / agent offline / no free executor |
| Fails before any stage, `GroovySandbox.parse` / `WorkflowScript: line X` | Jenkinsfile syntax error (unclosed quote, missing `}`). Read the **top** of the error for the line number |
| Fix made but Jenkins still fails the same way | Change not committed/pushed. Check `git status` and the commit ID in "Obtained Jenkinsfile from git ..." |
| `[Checks API] No suitable checks publisher found` | Harmless |
| Find a stage's error | Pipeline Overview → click the red box → look for `[ERROR]` lines |

---

## Still to do

1. Fix the current failing build
2. **Package** stage (`./mvnw -B package -DskipTests` → 8 jars)
3. **Docker image** stage (build each service with `docker/Dockerfile`)
4. **Trivy** image scan stage
5. **ngrok + GitHub webhook** (auto-trigger on push) and `waitForQualityGate`
6. **Email (SMTP) notifications** – Email Extension + Gmail app password, `post { failure / fixed }`
7. AWS credentials (later)
