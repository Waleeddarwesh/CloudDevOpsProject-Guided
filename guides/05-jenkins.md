# ⚙️ Module 05 — Jenkins

**Time:** ~5 hours

---

## 🎯 What you'll build

A **Groovy shared library** of 8 reusable steps powering a **9-stage pipeline**, consumed by three services. Each `Jenkinsfile` ends up ~15 lines.

```text
1 Checkout           tag = <build>-<git-sha>, never :latest
2 Unit Tests         containerised, per language
3 SonarQube          analysis + BLOCKING quality gate
4 Build Image        multi-stage, OCI provenance labels
5 Scan Image         Trivy — fixable CRITICAL ⇒ BUILD FAILS
6 Push Image         → ECR (IAM instance profile, no static keys)
7 Delete Image       reclaim disk
8 Update Manifests   kustomize edit set image
9 Push Manifests     git commit [skip ci] → ArgoCD deploys
```

---

# Part 1 · Jenkins web setup ✋

**This part cannot be automated safely** — Jenkins' first-run wizard is deliberately interactive.

### 1.1 Unlock

```bash
ssh -i ~/.ssh/ivolve-key.pem ubuntu@<JENKINS_IP> \
  'sudo cat /var/lib/jenkins/secrets/initialAdminPassword'
```

Open `http://<JENKINS_IP>:8080`, paste it.

**Select plugins → "Select none".** Ansible already installed 25 plugins in Module 03. Letting the wizard install its own set creates duplicates and version conflicts.

Create your admin user.

### 1.2 Register the shared library

**Manage Jenkins → System → Global Pipeline Libraries → Add**

| Field | Value |
|---|---|
| Name | `shared-library` |
| Default version | `main` |
| Retrieval | *Modern SCM* → *Git* |
| Repository | your shared-library repo URL |

> ⚠️ **Jenkins requires `vars/` at the repository ROOT.** It cannot load `05-Jenkins/vars/`. So the library lives in its own repo, mirrored from your project. That's a Jenkins constraint, not a design choice.

### 1.3 Credentials

**Manage Jenkins → Credentials → System → Global → Add**

| ID (must match **exactly**) | Kind | Value |
|---|---|---|
| `github-token` | Username with password | GitHub user + PAT |
| `sonar-token` | Secret text | from § 1.4 |

> ⚠️ These ID strings are referenced verbatim in the library. A typo produces `CredentialNotFoundException` **at runtime**, not at config time.

### 1.4 SonarQube — and the step everyone misses

Open `http://<JENKINS_IP>:9000` — `admin`/`admin`, change the password.

**a)** *My Account → Security → Generate Token* → add to Jenkins as `sonar-token`

**b)** *Manage Jenkins → System → SonarQube servers → Add*
- Name: `SonarQube`
- URL: `http://localhost:9000`
- Token: `sonar-token`

**c) ⚠️ THE WEBHOOK.** *SonarQube → Administration → Configuration → Webhooks → Create*
- Name: `Jenkins`
- URL: `http://localhost:8080/sonarqube-webhook/`

**Without this webhook, `waitForQualityGate()` blocks until it times out and every build hangs for 10 minutes before failing.** This is the single most commonly missed step in the entire project.

<details>
<summary><b>Why the webhook is required</b></summary>

The scanner uploads results **asynchronously** and exits immediately. Jenkins then asks SonarQube "did it pass?" — but SonarQube hasn't finished computing yet.

`waitForQualityGate()` pauses the pipeline and waits for SonarQube to **call back**. No webhook → no callback → the step waits until timeout.
</details>

---

# Part 2 · The shared library

## Layout

```text
vars/
├── microservicePipeline.groovy   the orchestrator — 9 stages
├── runUnitTests.groovy
├── sonarQubeScan.groovy
├── dockerBuildImage.groovy
├── trivyScan.groovy
├── ecrPush.groovy
├── updateManifests.groovy
└── pushManifests.groovy
```

Each file in `vars/` becomes a **global step** named after the file. `dockerBuildImage.groovy` defining `def call(Map)` gives you `dockerBuildImage(...)` in any pipeline.

## `trivyScan.groovy` — the security gate

```groovy
#!/usr/bin/env groovy
def call(Map config) {

    String image          = config.image
    boolean failOnCritical = config.failOnCritical != false
    String prefix         = config.reportPrefix ?: 'scan'

    // TWO invocations, deliberately.
    //
    // A single run with --exit-code 1 ABORTS before writing the report,
    // leaving you nothing to diagnose. So: scan once for the report,
    // scan again to gate.

    // --- Pass 1: always produce a report ---
    sh """
        trivy image \
          --format json \
          --output ${prefix}-trivy.json \
          --severity HIGH,CRITICAL \
          --ignore-unfixed \
          ${image} || true

        trivy image \
          --format table \
          --output ${prefix}-trivy.txt \
          --severity HIGH,CRITICAL \
          --ignore-unfixed \
          ${image} || true
    """

    archiveArtifacts artifacts: "${prefix}-trivy.*", allowEmptyArchive: true

    def counts = sh(
        script: """
            grep -o '"Severity":"CRITICAL"' ${prefix}-trivy.json | wc -l
        """,
        returnStdout: true
    ).trim()

    echo "Trivy: ${counts} fixable CRITICAL finding(s) in ${image}"

    // --- Pass 2: the gate ---
    if (failOnCritical) {
        sh """
            trivy image \
              --exit-code 1 \
              --severity CRITICAL \
              --ignore-unfixed \
              ${image}
        """
    }
}
```

### Two design decisions worth understanding

**`--ignore-unfixed`.** Without it, every build fails on unpatchable base-image CVEs that you cannot action. A gate everyone bypasses is worse than no gate — it trains people to ignore red builds.

**Fails on `CRITICAL` only.** HIGH is reported and archived; CRITICAL blocks. A calibrated threshold that people will actually respect.

## `ecrPush.groovy` — no credentials anywhere

```groovy
def call(Map config) {
    String image    = config.image
    String registry = config.registry
    String region   = config.region ?: 'us-east-1'

    // NOTE: no withCredentials(). The Jenkins EC2 instance carries an IAM
    // instance profile (Module 02), so the CLI obtains temporary,
    // auto-rotating credentials from the metadata service.
    //
    // No AKIA... key exists on this machine to leak.
    retry(3) {
        sh """
            aws ecr get-login-password --region ${region} \
              | docker login --username AWS --password-stdin ${registry}

            docker push ${image}
        """
    }
    // --password-stdin, never --password <token>. The latter puts the
    // secret in the process argument list, readable by `ps aux`.
    //
    // retry(3) is safe because a docker push is idempotent — layers
    // already present are skipped by digest.
}
```

## `updateManifests.groovy` — Kustomize, not sed

```groovy
def call(Map config) {
    String manifestDir = config.manifestDir
    String imageName   = config.imageName
    String newImage    = config.newImage

    if (!fileExists("${manifestDir}/kustomization.yaml")) {
        error("updateManifests: no kustomization.yaml in ${manifestDir}")
    }

    dir(manifestDir) {
        sh """
            set -eu
            kustomize edit set image ${imageName}=${newImage}
        """

        // Render the FULL output before anything is committed.
        // Catches a malformed kustomization HERE, in a failed build,
        // rather than in Git — where ArgoCD would mark the whole
        // Application unhealthy and block EVERY other service.
        sh "kubectl kustomize . > /dev/null && echo 'kustomize build OK'"
    }
}
```

> ⚠️ **You proved this in Module 04:** `kubectl kustomize .` renders **5** image lines and Kustomize rewrites exactly **3**. `sed -i 's|image: .*|image: NEW|g'` would rewrite all five — replacing `mysql:8.0` and `busybox:1.36` with an application image.

## `pushManifests.groovy` — the CI→CD handoff

```groovy
def call(Map config) {
    String gitRepo   = config.gitRepo        // github.com/User/Repo.git — NO scheme
    String gitBranch = config.gitBranch ?: 'main'

    if (gitRepo.startsWith('http')) {
        error("pushManifests: gitRepo must NOT include a scheme.")
    }

    withCredentials([usernamePassword(
        credentialsId: 'github-token',
        usernameVariable: 'GIT_USERNAME',
        passwordVariable: 'GIT_TOKEN'
    )]) {

        sh """
            git config user.email "jenkins@ivolve.io"
            git config user.name  "Jenkins CI"
            git add ${config.manifestDir}/kustomization.yaml

            if git diff --cached --quiet; then
                echo "No change — skipping commit."
                exit 0
            fi

            # ⚠️ [skip ci] stops this commit triggering the very pipeline
            # that created it. Without it, each build pushes a commit that
            # starts another build — an infinite loop that runs until
            # someone notices the executor is permanently busy.
            git commit -m "ci(${config.imageName}): deploy ${config.newImage}

            [skip ci]"
        """

        // Rebase-and-retry: three services push to one branch, so two
        // pipelines finishing together race and the loser is rejected
        // non-fast-forward.
        retry(3) {
            // NOTE the SINGLE quotes: ${GIT_TOKEN} is expanded by the
            // SHELL from an env var, not by Groovy. Groovy interpolation
            // would embed the secret in the script Jenkins writes to disk
            // and echoes to the console log.
            sh '''
                set -eu
                git pull --rebase origin ''' + gitBranch + ''' || true
                git push https://${GIT_USERNAME}:${GIT_TOKEN}@''' + gitRepo + ''' HEAD:''' + gitBranch + '''
            '''
        }
    }
}
```

> ⚠️ **Groovy vs shell interpolation is a real secret-leak vector.** `"${GIT_TOKEN}"` (double quotes) is substituted by Groovy *before* the script is written — so your token ends up in `/tmp` and in the build log. `'${GIT_TOKEN}'` (single quotes) is left for the shell.

## `microservicePipeline.groovy` — the orchestrator

```groovy
def call(Map config) {

    // Fail fast, BEFORE the pipeline starts. One clear message beats a
    // NullPointerException fifteen minutes into a build.
    List required = ['serviceName', 'sourceDir', 'language', 'ecrRegistry', 'gitRepo']
    List missing  = required.findAll { !config.containsKey(it) || !config[it] }
    if (missing) {
        error("microservicePipeline: missing required parameter(s): ${missing.join(', ')}")
    }

    pipeline {
        agent any

        options {
            timeout(time: 30, unit: 'MINUTES')
            buildDiscarder(logRotator(numToKeepStr: '20'))
            disableConcurrentBuilds()      // prevents the manifest push race
            timestamps()
            skipDefaultCheckout(true)      // so cleanWs() can run first
        }

        environment {
            SERVICE_NAME = "${config.serviceName}"
            ECR_REGISTRY = "${config.ecrRegistry}"
            MANIFEST_DIR = "${config.manifestDir ?: '04-Kubernetes/manifests'}"
        }

        stages {
            stage('Checkout') {
                steps {
                    cleanWs()
                    script {
                        def scmVars = checkout scm
                        env.GIT_COMMIT_SHORT = scmVars.GIT_COMMIT.take(7)

                        // ⚠️ IMMUTABLE TAG. Never :latest.
                        //
                        // :latest means the image running in production is
                        // unidentifiable, rollback is impossible, and the
                        // thing you scanned may not be the thing you shipped.
                        env.IMAGE_TAG  = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                        env.FULL_IMAGE = "${env.ECR_REGISTRY}/${env.SERVICE_NAME}:${env.IMAGE_TAG}"

                        currentBuild.displayName = "#${BUILD_NUMBER} · ${SERVICE_NAME}"
                    }
                }
            }

            stage('Unit Tests')  { steps { script { runUnitTests(language: config.language, sourceDir: config.sourceDir) } } }

            stage('SonarQube Analysis') {
                when { expression { config.runSonar != false } }
                steps { script { sonarQubeScan(projectKey: config.serviceName, sourceDir: config.sourceDir, language: config.language) } }
            }

            stage('Build Image') { steps { script { dockerBuildImage(image: env.FULL_IMAGE, context: config.sourceDir) } } }
            stage('Scan Image')  { steps { script { trivyScan(image: env.FULL_IMAGE, failOnCritical: config.failOnCritical != false, reportPrefix: config.serviceName) } } }
            stage('Push Image')  { steps { script { ecrPush(image: env.FULL_IMAGE, registry: env.ECR_REGISTRY) } } }

            stage('Delete Image Locally') {
                // 3 services × ~400 MB × 20 builds exhausts a 50 GB volume
                // in days. The image is safely in ECR by now.
                steps { sh "docker rmi ${env.FULL_IMAGE} || true" }
            }

            stage('Update Manifests') {
                steps { script { updateManifests(manifestDir: env.MANIFEST_DIR, imageName: env.SERVICE_NAME, newImage: env.FULL_IMAGE) } }
            }

            stage('Push Manifests') {
                steps { script { pushManifests(manifestDir: env.MANIFEST_DIR, imageName: env.SERVICE_NAME, newImage: env.FULL_IMAGE, gitRepo: config.gitRepo, gitBranch: config.gitBranch ?: 'main') } }
            }
        }

        post {
            success { echo "✅ ${env.FULL_IMAGE} pushed. ArgoCD will sync." }
            failure { echo "❌ FAILED at stage '${env.STAGE_NAME}'" }
            always  { cleanWs() }
        }
    }
}
```

---

# Part 3 · The Jenkinsfiles

`workspace/05-Jenkins/Jenkinsfiles/frontend.Jenkinsfile`:

```groovy
// The trailing `_` is MANDATORY. It is not a typo.
// @Library must attach to a statement, and `_` is the conventional no-op.
// Omit it and you get "unable to resolve class" with no hint why.
@Library('shared-library') _

microservicePipeline(
    serviceName: 'ivolve-frontend',
    sourceDir:   'src/frontend',
    language:    'node',
    ecrRegistry: '123456789012.dkr.ecr.us-east-1.amazonaws.com',
    gitRepo:     'github.com/YourUser/CloudDevOpsProject.git'
)
```

Repeat for `auth-service` (`language: 'python'`) and `roadmap-service` (`language: 'java'`).

**Fifteen lines per service. All the logic is in the library.** That's the point of Lab 23.

---

# Part 4 · Create the jobs

For each service — **New Item → Pipeline**:

- **Definition:** *Pipeline script from SCM*
- **SCM:** Git → your repo → credentials `github-token`
- **Branch:** `*/main`
- **Script Path:** `05-Jenkins/Jenkinsfiles/frontend.Jenkinsfile`

Set the real account ID first:

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/123456789012/$ACCOUNT/g" 05-Jenkins/Jenkinsfiles/*.Jenkinsfile
git commit -am "chore: real AWS account ID" && git push
```

---

# ✅ Checkpoint

Build all three. Watch the stage view.

```bash
aws ecr list-images --repository-name ivolve-frontend \
  --query 'imageIds[*].imageTag' --output table
```

```text
|  42-a1b2c3d  |
```

**A build number and a git SHA. Not `latest`.** You can trace that exact image back to the commit that produced it.

Then check Git:

```bash
git pull
git log --oneline -3
```

```text
a1b2c3d ci(ivolve-frontend): deploy ...ivolve-frontend:42-a1b2c3d
```

**Jenkins committed to your repo.** That commit is the CI→CD handoff — Module 06 makes ArgoCD act on it.

---

## 🔨 Deliberately break the security gate

Worth doing once, to see the gate work:

```groovy
microservicePipeline(
    // ...
    failOnCritical: true
)
```

Temporarily change a Dockerfile to an old base image with known CRITICALs:

```dockerfile
FROM node:18.0-alpine        # old, vulnerable
```

Run the build. **Stage 5 fails and no image reaches ECR.** Read the archived `*-trivy.txt` report — it lists each CVE and the version that fixes it.

Revert afterwards. **You just proved a vulnerable image cannot reach your registry.**

---

## 🧠 Before you continue

1. Why is the trailing `_` after `@Library` required?
2. What does the SonarQube **webhook** fix?
3. Why does `trivyScan` run Trivy twice?
4. Why `--ignore-unfixed`?
5. Why does `ecrPush` need no credentials?
6. What does `[skip ci]` prevent?
7. Why single quotes around `${GIT_TOKEN}` in the shell block?
8. Why `<build>-<sha>` instead of `:latest`?

---

<div align="center">

**Next → [Module 06 — ArgoCD](06-argocd.md)**

</div>
