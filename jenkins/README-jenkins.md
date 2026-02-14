# Phase 1 — Setting Up Jenkins Locally (Docker)
## Objective

The goal of this phase is to:

- Run Jenkins locally using Docker

- Connect Jenkins to this GitHub repository

- Execute a simple pipeline

- Confirm Jenkins can run Docker commands

This ensures our CI system is correctly configured before moving to AWS deployment.

##
### Architecture (Phase 1)

```nginx
Developer → GitHub → Jenkins (Docker on local machine)
```
At this stage:

- Jenkins runs inside a Docker container

- Jenkins reads the `Jenkinsfile` from this repository

- Jenkins executes pipeline steps locally

No AWS or deployment is involved yet.
##
### Step 1 — Run Jenkins in Docker

Create a persistent volume:
```bash
docker volume create jenkins_home
```

### What is a Docker volume?

A Docker volume is:

    Persistent storage managed by Docker.

Normally, when you stop and delete a container, all its data is lost.

But Jenkins stores:

- Installed plugins

- Job configurations

- Credentials

- Build history

- Pipeline settings

If we didn’t use a volume:

- Every time Jenkins restarts, it would reset.

- We would lose everything.

Run jenkins:
```bash
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -u root -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins:lts
```
**really long let me breakdown for you :**

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -u root \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  my-jenkins-docker

```
**What `-u` root does**

This flag runs the Jenkins container as the root user.

Root has permission to access:
```arduino
/var/run/docker.sock
```
This allows Jenkins to execute:

- docker build

- docker run

- docker push

- docker compose

### Why This Is Not Suitable for Production

Running CI/CD systems as root is not recommended in production environments because:

- It bypasses Linux security controls

- It increases the attack surface

- If Jenkins is compromised, the attacker gains root-level access to Docker

- Docker socket access effectively grants host-level control

In enterprise environments, this would be considered a security risk.

`-p 8080:8080`

This means:

- Host machine port 8080

- Connects to container port 8080

so when you open:

```arduino
http://localhost:8080
```

you are accesing:

```scss
Jenkins running inside the container
```


`-p 50000:50000`

This port is used by:

- Jenkins agents (distributed builds)

- Future scaling if needed

For now, it’s not heavily used, but it’s best practice to expose it.
##
`-v jenkins_home:/var/jenkins_home`

This provides:

    Persistent storage for Jenkins data.

Without it:

- Restart = data loss

- Reinstall plugins every time

- Recreate jobs every time

This is critical.
##
`-v /var/run/docker.sock:/var/run/docker.sock`

This one is extremely important.

What is `/var/run/docker.sock`?

It is:

    The communication socket between Docker CLI and Docker daemon.

By mounting it into the Jenkins container:

We allow Jenkins to:

- Build Docker images

- Run containers

- Push to Docker Hub

- Use Docker Compose

Without this mount:
```bash
docker build ...
```

it wil fail with:

```pgsql
Cannot connect to Docker daemon
```

## Visual Explanation

```arduino
Your Laptop
│
├── Docker Engine
│     ├── Jenkins Container
│     │      ├── Jenkins UI
│     │      └── Can run docker commands
│     │
│     └── Docker Volume: jenkins_home
│
└── Browser → localhost:8080 → Jenkins
```
##
### Summary Table
| Flag                                           | Purpose                         | Why Needed                |
| ---------------------------------------------- | ------------------------------- | ------------------------- |
| `-p 8080:8080`                                 | Exposes Jenkins UI              | Access via browser        |
| `-p 50000:50000`                               | Jenkins agent communication     | Future scalability        |
| `-v jenkins_home:/var/jenkins_home`            | Persistent data storage         | Prevent data loss         |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Allow Jenkins to control Docker | Build & deploy containers |


configuration for your first job to chek it work:
- select new job
- type: pipeline 
![img](../img-documentation/new-jenkins-doc.png)

# PHASE 2 — Build Sparta App Docker Image with Jenkins (Clean Start)

### STEP 1 — Confirm Project Structure

Your repository should look like this:

```csharp
sparta-with-jenkins/
│
├── app/
│   ├── Dockerfile
│   ├── package.json
│   ├── app.js
│   ├── seeds/
│   ├── models/
│   └── ...
│
└── jenkins/
    └── Jenkinsfile
```
Important:

- `Dockerfile` must be inside `app/`

- It must be named exactly: `Dockerfile` (capital D)

- No extension
##
### STEP 2 —  Dockerfile 

So your app/Dockerfile must be:

```dockerfile
FROM node:20-alpine

WORKDIR /usr/src/app

# Copy everything first (so seeds exist)
COPY . .

# Then install dependencies
RUN npm install

EXPOSE 3000

CMD ["node", "app.js"]
```
commit the changes:
```bash
git add .
git commit -m " Dockerfile build order"
git push
```
##
### STEP3 - Clean Jenkinsfile
Replace your `jenkins/Jenkinsfile` with this minimal clean version:
```groovy
pipeline {
  agent any

  environment {
    IMAGE_NAME = "jrodga1604/sparta-app"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        dir('app') {
          sh 'docker build -t $IMAGE_NAME:latest .'
        }
      }
    }

  }
}
```

once again commit and push

```bash
git add .
git commit -m "Phase 2 clean build pipeline"
git push
```
##
### STEP 5 — Run Pipeline

Click:

Build Now

Watch console.

You should see:
```bash
Step 1/...
Successfully built ...
Successfully tagged jrodga1604/sparta-app:latest
```
![img](../img-documentation/succes-first-job.png)

##
### STEP 6 — Verify on Host

Run on your machine:
```bash
docker images
```
![img](<../img-documentation/Screenshot 2026-02-12 at 17.30.44.png>)

# PHASE 3 — Push Docker Image to Docker Hub

### Objective

After this phase:

- Every push to main

- Jenkins builds your image

- Jenkins logs in to Docker Hub securely

- Jenkins pushes the image

- Image becomes available globally

Architecture now becomes:

```markdown
Developer → GitHub → Jenkins
                          ↓
                   docker build
                          ↓
                    docker push
                          ↓
                     Docker Hub
```
##
### STEP 1 — Create Docker Hub Access Token

Go to Docker Hub:

Account Settings → Security → Access Tokens → New Token

Create:
```mathematica
Name: jenkins-ci
Permissions: Read & Write
```
![img](<../img-documentation/Screenshot 2026-02-12 at 17.45.24.png>)
**Copy the token.**
##
### STEP 2 — Add Docker Hub Credentials in Jenkins

Go to:

Manage Jenkins → Credentials → Global → Add Credentials

Choose:

- Kind: Username with password

- Username: your Docker Hub username (e.g. jrodga1604)

- Password: paste Docker Hub token

- ID: dockerhub-creds

- Description: Docker Hub CI

**Save.**
![img](../img-documentation/token.png)
##
### STEP 3 — Update Jenkinsfile

Replace your current Jenkinsfile with this:

```groovy
pipeline {
  agent any

  environment {
    IMAGE_NAME = "jrodga1604/sparta-app"
    DOCKER_CREDS = "dockerhub-creds"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        dir('app') {
          sh """
            docker build -t $IMAGE_NAME:$BUILD_NUMBER .
            docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest
          """
        }
      }
    }

    stage('Login to Docker Hub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: DOCKER_CREDS,
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
        }
      }
    }

    stage('Push Image') {
      steps {
        sh """
          docker push $IMAGE_NAME:$BUILD_NUMBER
          docker push $IMAGE_NAME:latest
        """
      }
    }

  }
}


```

### What Changed?

We introduced:
```scss
withCredentials(...)
```
his tells Jenkins:

- Retrieve stored credentials securely

- Inject them as temporary environment variables

- Use them only inside this block

### Why We Did This

Docker Hub requires authentication before pushing images.

Instead of writing:
```bash
docker login -u myusername -p mypassword
```

(which would expose credentials in logs)

We use Jenkins Credentials Store.

This ensures:

- Password is encrypted

- It is never committed to Git

- It never appears in console output

- It follows secure CI/CD practice

##
### New Stage Added — Push Image

We added:
```groovy
stage('Push Image') {
  steps {
    sh 'docker push $IMAGE_NAME:latest'
  }
}
```
### What Changed?

Previously, we only built:
```nginx
docker build
```

Now we also push:
```perl
docker push
```
### Why We Did This

The purpose of CI/CD is not just building.

It is:

- Producing a deployable artifact

- Storing it in a registry

- Making it available for production systems

 By pushing to Docker Hub:

- The image becomes globally accessible

- EC2 can pull it

- Kubernetes can pull it

Future environments can use it

### What we improved
```perl
docker push image:$BUILD_NUMBER
docker push image:latest
```

We updated the Jenkins pipeline to tag images using:

- The Jenkins BUILD_NUMBER (immutable)

- The latest tag (for convenience)

```bash
jrodga1604/sparta-app:12
jrodga1604/sparta-app:latest
```
Now:

- Each build produces a uniquely versioned image

- Previous builds remain available

- Rollback is possible

- Deployment becomes reproducible

### Why this is important in CI/CD

Using build numbers provides:

- Traceability (which pipeline produced which image)

- Stability (deploy specific versions)

- Better debugging

- Safer production releases

This follows industry-standard CI/CD practices where artifacts are immutable and versioned.

now **Commit & push.**
##
### STEP 4 — Run Pipeline

Click **Build Now**

You should see:
```markfile
Login Succeeded
Pushing...
latest: digest: sha256:...
```
##
### STEP 5 — Verify on Docker Hub

Go to your Docker Hub repo:
```bash
https://hub.docker.com/r/jrodga1604/sparta-app
```

You should see:
```nginx
latest
```

Updated recently.


##
### Phase 3B — Automated CI for Feature Branches
**Objective**

This stage introduces a second Jenkins job dedicated to Continuous Integration (CI) for feature branches.

The purpose is to:

- Automatically build feature branches

- Validate changes before merging to main

- Prevent unstable code from reaching production

- Simulate enterprise CI workflow
##
### CI Architecture Overview

```pgsql
Developer → Push feature branch
        ↓
GitHub Webhook
        ↓
ngrok tunnel
        ↓
Jenkins (local)
        ↓
Build & Validate
```
##
### Why a Second Job Was Created

Instead of modifying the original pipeline job, a second job was created:
 
 | Job Name             | Purpose                                  | Trigger        |
| -------------------- | ---------------------------------------- | -------------- |
| `sparta-ci-feature`  | Automatic validation of feature branches | GitHub Webhook |
| `sparta-manual-main` | Manual testing & controlled builds       | Manual         |

This separation ensures:

- Clear responsibility per job

- Safer deployment logic

- Better CI/CD structure

- Easier debugging

This mirrors real-world enterprise CI design.

##
### Jenkins Job Configuration — `sparta-ci-feature`
1. **Create Pipeline Job**

- Name: `sparta-ci-feature`

- Type: Pipeline

- Definition: Pipeline script from SCM

- SCM: Git

- Repository: Your GitHub repo

2. **Branch Specifier**

```bash
*/feature-*
```
This tells Jenkins:

- Only build branches that start with `feature-`

3. **Enable Automatic Trigger**

Under **Build Triggers**, enable:

```rust
GitHub hook trigger for GITScm polling
```
This allows Jenkins to respond to GitHub webhook events.
##
### Exposing Local Jenkins Using ngrok

Since Jenkins runs locally, GitHub cannot access:
```arduino
http://localhost:8080
```
To solve this, we use **ngrok**, which creates a public tunnel to the local machine.

### Install ngrok
Mac:
```bash
brew install ngrok
```
Or download from:

https://ngrok.com/download

##
### Authenticate ngrok

After creating an ngrok account, run:
```
ngrok config add-authtoken YOUR_AUTH_TOKEN
```
### Start ngrok Tunnel

Run:
```bash
ngrok http 8080
```

You will see:
```nginx
Forwarding https://abcd-1234.ngrok.io -> http://localhost:8080
```
![img](<../img-documentation/Screenshot 2026-02-13 at 08.29.39.png>)
Copy the HTTPS URL.
##
### Configure GitHub Webhook

In your GitHub repository:

1. Go to **Settings** → Webhooks

2. Click **Add webhook**

Set:

Payload URL:
```bash
https://abcd-1234.ngrok.io/github-webhook/
```

Important: include `/github-webhook/`

Content type:
```bash
application/json
```

Events:
```csharp
Just the push event
```

**Save.**

Testing the Automatic CI Job

Create and push a feature branch:
```bash
git checkout -b feature-ci-test
# make a small change
git add .
git commit -m "Testing CI auto trigger"
git push -u origin feature-ci-test
```

Expected behavior:

- GitHub sends webhook

- ngrok forwards request

- Jenkins triggers sparta-ci-feature

- Pipeline builds automatically

- No manual action required
##
### Why This Approach Is Important

This setup introduces real Continuous Integration behavior:

- Developers do not manually trigger builds

- Every feature branch is validated

- Broken builds are caught early

- Production remains protected

This mirrors professional DevOps environments where:

- Feature branches trigger CI pipelines

- Main branch triggers deployment pipelines

### Important Notes About ngrok

- Free ngrok URLs change every restart

- If ngrok restarts, webhook URL must be updated

- ngrok must remain running while testing

For production environments:

- Jenkins should run on a public server (e.g., AWS EC2)

- A proper domain and HTTPS certificate should be configured

ngrok is used here for local development convenience.

##
# DIAGRAMS OF THE ARCHITECTURE:

### CI/CD Architecture Overview
```pgsql
                         ┌─────────────────────────┐
                         │       Developer         │
                         │  git push feature-*     │
                         └─────────────┬───────────┘
                                       │
                                       ▼
                         ┌─────────────────────────┐
                         │        GitHub           │
                         │   (Source Control)      │
                         └─────────────┬───────────┘
                                       │ Webhook
                                       ▼
                         ┌─────────────────────────┐
                         │        ngrok Tunnel     │
                         │ (Public → Local Jenkins)│
                         └─────────────┬───────────┘
                                       ▼
                         ┌─────────────────────────┐
                         │     Jenkins (Local)     │
                         │  Job: sparta-ci-feature │
                         │  - Checkout branch      │
                         │  - Build Docker image   │
                         │  - Push to Docker Hub   │
                         └─────────────┬───────────┘
                                       ▼
                         ┌─────────────────────────┐
                         │      Docker Hub         │
                         │  Image tagged with      │
                         │  build number + latest  │
                         └─────────────────────────┘
```

### Branch Strategy Diagram
```pgsql
feature-login  ─┐
feature-api     ├──► CI Job (Build + Validate Only)
feature-ui      ┘

main ───────────────► Manual Job (Build + Push + Deploy)

```

##
# PHASE 4 — Automated Deployment to EC2

### — Architecture Compatibility Warning
### Important — Docker Architecture Compatibility (Apple Silicon vs AWS EC2)

During deployment testing, the following error may appear on EC2:
```bash
no matching manifest for linux/amd64 in the manifest list entries
```
Why This Happens

If Jenkins is running on:

- Apple Silicon (M1/M2 → arm64)

And AWS EC2 is running:
```bash
linux/amd64
```

Then a normal Docker build command:
```bash
docker build ...
```

Will produce an arm64 image, which EC2 cannot run.

Docker builds images for the architecture of the machine running the build.

This causes a platform mismatch.
##
### Fix — Build for EC2 Architecture Explicitly

To ensure compatibility, the Jenkins pipeline must build the image for:
```bash
linux/amd64
```

This is done using Docker Buildx.
##

### Jenkinsfile Modification (Phase 4 Update)
* Previous Build Stage
```groovy
stage('Build Docker Image') {
  steps {
    dir('app') {
      sh """
        docker build -t $IMAGE_NAME:$BUILD_NUMBER .
        docker tag $IMAGE_NAME:$BUILD_NUMBER $IMAGE_NAME:latest
      """
    }
  }
}
```
Problem:

- Builds only for local architecture

- Causes deployment failure on EC2

* Updated Cross-Platform Build Stage
```groovy
stage('Build and Push Docker Image (amd64)') {
  steps {
    dir('app') {
      sh """
        docker buildx create --use --name multi-builder || true
        docker buildx build \
          --platform linux/amd64 \
          -t $IMAGE_NAME:$BUILD_NUMBER \
          -t $IMAGE_NAME:latest \
          --push .
      """
    }
  }
}
```
##
### What changed
| Change                         | Reason                             |
| ------------------------------ | ---------------------------------- |
| Replaced `docker build`        | Enables multi-platform support     |
| Added `--platform linux/amd64` | Ensures compatibility with AWS EC2 |
| Added `--push`                 | Required by buildx to push images  |
| Removed separate push stage    | No longer necessary                |
##
### Why This Is Important

Cloud servers (AWS EC2) typically use:
```bash
linux/amd64
```

Modern laptops (especially Apple Silicon) use:
```bash
linux/arm64
```

Without explicitly setting the platform, Docker images may not run in cloud environments.

This update ensures:

- Architecture compatibility

- Reliable deployment

- Production-ready CI/CD behavior

### Goal

After Phase 4:

When code is merged to main:

- Jenkins builds image

- Pushes to Docker Hub

- Connects to EC2 via SSH

- Pulls latest image

- Restarts container automatically

This becomes real Continuous Deployment.
##
### Final Architecture After Phase 4
```arduino
Developer → GitHub → Jenkins
                          ↓
                     Docker Hub
                          ↓
                        EC2
                 (Docker running)
                          ↓
                  Sparta App Live
```
##
### STEP 1 — Prepare EC2 Instance

Launch EC2:

- Ubuntu 22.04

- t3.micro (free tier)

- Open ports:

    - 22 (SSH)

    - 3000 (app)

![img](../img-documentation/ec2-sg.png)

- I use this from anywhere because it is experimental and for practice purposes, **not suitable for production.**
### Install Docker on EC2
Get inside of your new Ec2 instance with SSH
```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
sudo apt install docker-compose -y
```
Add ubuntu user to docker group:
```bash
sudo usermod -aG docker ubuntu
```
Log out and back in.
##
### Test Docker
```bash
docker --version
```
![img](../img-documentation/docker-version.png)
##
### STEP 2 — Create docker-compose.yml on EC2

On EC2:
```bash
nano docker-compose.yml
```
![img](../img-documentation/nano-document.png)

##
### STEP 3 — Test Manual Deployment
On EC2:
```bash
docker pull jrodga1604/sparta-app:latest
docker compose up -d
docker ps
```
![img](../img-documentation/docker-runing.png)

Visit:
```cpp
http://EC2_PUBLIC_IP:3000
```

If working → EC2 is ready.
![img](../img-documentation/sparta-working-ec2.png)
##
### STEP 4 — Add SSH Credentials in Jenkins

Go to:

Manage Jenkins → Credentials → Global → Add Credentials

Kind:
```vbnet
SSH Username with private key
```

- Username: ubuntu

- Private key: paste EC2 .pem key

- ID: ec2-ssh

- Add the value of your pem key (**DO NOT SHARE**)
![img](../img-documentation/jenkis-shh.png)

**Save.**

##
### STEP 5 — Update Jenkinsfile (Main Job Only)

In your main branch job, add:
```groovy
stage('Deploy to EC2') {
  when {
    branch 'main'
  }
  steps {
    sshagent(credentials: ['ec2-ssh']) {
      sh """
        ssh -o StrictHostKeyChecking=no ubuntu@YOUR_EC2_IP '
          docker pull jrodga1604/sparta-app:latest &&
          docker compose down &&
          docker compose up -d
        '
      """
    }
  }
}
```

Replace:
```nginx
YOUR_EC2_IP
```
##

### What This Does

When main builds:

1. Jenkins logs into EC2

2. Pulls latest image

3. Stops running container

4. Starts new container

Fully automated deployment.

You’ve already noticed that hardcoding the EC2 IP isn’t a good practice, but this will be resolved when we move to Kubernetes in the next steps.

## 
### Before we move to Kubernetes
### Add one job more that will merge the branches if the previus is success 

```css
️ Push to feature branch
        ↓
Job 1 → CI (build & test only)
        ↓ (if SUCCESS)
️ Job 2 → Merge feature → main
        ↓
Deploy new main

```

### Important Before Doing This

Make sure:

- Job 1 builds only feature-*

- Job 2 builds only main

- Jenkins has GitHub credentials with push access

Otherwise it won’t be able to merge.

### Add GitHub Credentials in Jenkins

Manage Jenkins → Credentials → Global → Add
![img](../img-documentation/token-github.png)
Type:
```nginx
Username + Personal Access Token
```

ID:
```
github-creds
````

You need a GitHub Personal Access Token with:
``` nginx
repo
```
![img](../img-documentation/repo.png)
permissions.
##
### Modify Job 1 Jenkinsfile (CI Feature)

We add merge stage at the end.

Here is the updated version:

```groovy
pipeline {
  agent any

  environment {
    IMAGE_NAME = "jrodga1604/sparta-app"
    GIT_CREDS = "github-creds"
    EC2_IP = "YOUR_EC2_PUBLIC_IP"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        dir('app') {
          sh """
            docker buildx create --use --name multi-builder || true
            docker buildx build \
              --platform linux/amd64 \
              -t $IMAGE_NAME:$BUILD_NUMBER \
              -t $IMAGE_NAME:latest \
              --push .
          """
        }
      }
    }

    stage('Merge to Main (Promotion Job Only)') {
      when {
        expression { env.JOB_NAME == "sparta-promote-main" }
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: GIT_CREDS,
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh """
            git config user.email "jenkins@local"
            git config user.name "Jenkins"

            git checkout main
            git pull https://$GIT_USER:$GIT_TOKEN@github.com/jrodga/sparta-with-jenkins.git main

            git merge origin/${BRANCH_NAME}

            git push https://$GIT_USER:$GIT_TOKEN@github.com/jrodga/sparta-with-jenkins.git main
          """
        }
      }
    }

    stage('Deploy to EC2 (Main Only)') {
      when {
        branch 'main'
      }
      steps {
        sshagent(credentials: ['ec2-ssh']) {
          sh """
            ssh -o StrictHostKeyChecking=no ubuntu@$EC2_IP '
              docker pull $IMAGE_NAME:latest &&
              docker compose down &&
              docker compose up -d
            '
          """
        }
      }
    }

  }
}
```
last update!