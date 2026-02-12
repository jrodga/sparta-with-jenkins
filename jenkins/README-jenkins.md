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
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins:lts
```
**really long let me breakdown for you :**

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

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