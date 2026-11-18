# Docker & AWS ECR --- Hands-on Practice Lab

A practical learning repository documenting Docker on Debian Linux,
container lifecycle management, networking, volumes, custom images,
Docker Compose, a Node.js + MongoDB application, and publishing images
to Amazon Elastic Container Registry (ECR).

> **Lab goal:** Understand the full container workflow: run existing
> images, connect containers, persist data, build an application image,
> orchestrate services with Compose, and publish an image to ECR.

------------------------------------------------------------------------

## Table of contents

1.  [Lab architecture](#lab-architecture)
2.  [Prerequisites](#prerequisites)
3.  [Install and verify Docker on
    Debian](#1-install-and-verify-docker-on-debian)
4.  [Docker fundamentals](#2-docker-fundamentals)
5.  [Container lifecycle](#3-container-lifecycle)
6.  [Docker networking](#4-docker-networking)
7.  [Docker volumes](#5-docker-volumes)
8.  [Build a custom Node.js image](#6-build-a-custom-nodejs-image)
9.  [Run MongoDB and Mongo Express
    manually](#7-run-mongodb-and-mongo-express-manually)
10. [Docker Compose](#8-docker-compose)
11. [Node.js application and MongoDB
    connection](#9-nodejs-application-and-mongodb-connection)
12. [Publish an image to AWS ECR](#10-publish-an-image-to-aws-ecr)
13. [Useful troubleshooting commands](#11-troubleshooting)
14. [Security notes](#12-security-notes)
15. [Practice checklist](#13-practice-checklist)

------------------------------------------------------------------------

## Lab architecture

The final lab contains three services on one Compose-managed network:

``` text
                    Debian host
                         |
             +-----------+-----------+
             |                       |
       Node.js app              Mongo Express
       host port 3000            host port 8080
             |                       |
             +-----------+-----------+
                         |
                 Compose network
                         |
                    MongoDB
                  container port
                      27017
                         |
                  Named volume
                    mongo-data
```

  Component       Purpose                                                       Host port
  --------------- --------------------------------------- -------------------------------
  Node.js app     Demo web application                                             `3000`
  MongoDB         Database                                  `27017` (optional to publish)
  Mongo Express   Browser-based database administration                            `8080`

Inside the Docker network, services connect to MongoDB using the
hostname `mongodb` and port `27017`. A container should not use
`localhost` to reach another container.

------------------------------------------------------------------------

## Prerequisites

-   Debian Linux
-   Docker Engine
-   Docker Compose V2 plugin (`docker compose`, with a space)
-   Git
-   Node.js and npm only if running the app directly on the Debian host
-   AWS CLI only for the ECR section
-   An AWS identity with the required ECR permissions

Check installed versions:

``` bash
docker --version
docker compose version
git --version
aws --version
```

If `docker compose version` fails, install the Compose plugin using the
package source appropriate to your Docker installation. On Debian
systems using Docker's official repository:

``` bash
sudo apt update
sudo apt install docker-compose-plugin
```

If the package is unavailable, check that the Docker repository is
configured correctly rather than installing random binaries.

------------------------------------------------------------------------

## 1. Install and verify Docker on Debian

Installation instructions can differ depending on whether Docker came
from Debian's repository or Docker's official repository. Follow the
official Docker Engine for Debian instructions for a fresh installation.

After installation:

``` bash
sudo systemctl status docker
sudo systemctl enable --now docker
sudo docker run hello-world
```

Optional: allow your regular user to run Docker without `sudo`:

``` bash
sudo usermod -aG docker "$USER"
```

Log out and back in for the group change to take effect. Membership in
the `docker` group effectively grants root-level privileges, so only
trusted users should be added.

------------------------------------------------------------------------

## 2. Docker fundamentals

### Images

Images are templates used to create containers.

``` bash
# Download an image
docker pull nginx

# List local images
docker images

# Inspect an image
docker image inspect nginx

# Remove an image (only if no container depends on it)
docker rmi nginx
```

### Run a first container

``` bash
# Run nginx in detached mode and publish host port 8081 to container port 80
docker run -d --name web-test -p 8081:80 nginx

# Open http://localhost:8081 from the Docker host

# Show running containers
docker ps

# Show all containers, including stopped ones
docker ps -a
```

------------------------------------------------------------------------

## 3. Container lifecycle

``` bash
# Stop a running container
docker stop web-test

# Start an existing stopped container
docker start web-test

# Restart a container
docker restart web-test

# View container logs
docker logs web-test

# Follow logs continuously
docker logs -f web-test

# Inspect container configuration
docker inspect web-test

# Execute a command inside a running container
docker exec -it web-test sh

# View live CPU and memory usage
docker stats

# Remove a stopped container
docker rm web-test

# Force-remove a running container (use carefully)
docker rm -f web-test
```

A container name must be unique. If Docker reports that a name is
already in use, inspect the existing container and either reuse it,
remove it, or choose a different name.

------------------------------------------------------------------------

## 4. Docker networking

Containers on a user-defined bridge network can discover each other by
container name or network alias.

``` bash
# Create a network
docker network create mongo-network

# List networks
docker network ls

# Inspect a network
docker network inspect mongo-network

# Run MongoDB on the network
docker run -d \
  --name mongodb \
  --network mongo-network \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD='change-this-password' \
  mongo

# Run Mongo Express on the same network
docker run -d \
  --name mongo-express \
  --network mongo-network \
  -p 8081:8081 \
  -e ME_CONFIG_MONGODB_URL='mongodb://admin:change-this-password@mongodb:27017/?authSource=admin' \
  -e ME_CONFIG_BASICAUTH_USERNAME=admin \
  -e ME_CONFIG_BASICAUTH_PASSWORD='change-this-web-password' \
  mongo-express

# Check that the MongoDB service name resolves from Mongo Express
docker exec mongo-express getent hosts mongodb

# Test TCP connectivity from Mongo Express (if bash is available)
docker exec mongo-express bash -c 'echo > /dev/tcp/mongodb/27017 && echo CONNECTED'

# Inspect network membership
docker network inspect mongo-network

# Remove containers before removing their network
docker rm -f mongo-express mongodb
docker network rm mongo-network
```

**Key lesson:** the Mongo Express connection hostname must match the
MongoDB container name or network alias. If the MongoDB container is
named `mongodb`, use `mongodb`, not `mongo`.

------------------------------------------------------------------------

## 5. Docker volumes

A container's writable layer is not a reliable place to keep important
database data. Use a volume for persistent storage.

``` bash
# Create a named volume
docker volume create mongo-data

# List volumes
docker volume ls

# Inspect a volume
docker volume inspect mongo-data

# Run MongoDB with persistent storage
docker run -d \
  --name mongodb \
  --network mongo-network \
  -v mongo-data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD='change-this-password' \
  mongo

# Remove a volume (DELETES its stored data)
docker volume rm mongo-data
```

**Warning:** `docker compose down -v` removes Compose-managed named
volumes and their data. Do not use it when you need to preserve the
database.

------------------------------------------------------------------------

## 6. Build a custom Node.js image

A Dockerfile describes how to build an image. Example for a Node.js
application whose `package.json` has a valid start script or whose entry
point is `server.js`:

``` dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy dependency manifests first to improve build-cache use
COPY package*.json ./

# Use npm ci when package-lock.json is committed
RUN if [ -f package-lock.json ]; then npm ci --omit=dev; else npm install --omit=dev; fi

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

Create a `.dockerignore` file next to the Dockerfile:

``` text
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
```

Build and inspect the image:

``` bash
# Run from the directory containing the Dockerfile
docker build -t node-profile-app:1.0 .

# List images
docker images

# Inspect image metadata
docker image inspect node-profile-app:1.0

# Run the app (assuming it listens on 0.0.0.0:3000)
docker run -d \
  --name node-app-test \
  -p 3000:3000 \
  node-profile-app:1.0

# View logs
docker logs -f node-app-test

# Stop and remove the test container
docker rm -f node-app-test
```

The application must listen on `0.0.0.0` inside its container, not only
on `127.0.0.1` or `localhost`.

------------------------------------------------------------------------

## 7. Run MongoDB and Mongo Express manually

The following is a learning example. Use a strong, unique password in a
real environment.

``` bash
# Create the network
docker network create mongo-network

# Start MongoDB
docker run -d \
  --name mongodb \
  --network mongo-network \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD='change-this-password' \
  -v mongo-data:/data/db \
  mongo

# Start Mongo Express
docker run -d \
  --name mongo-express \
  --network mongo-network \
  -p 8081:8081 \
  -e ME_CONFIG_MONGODB_URL='mongodb://admin:change-this-password@mongodb:27017/?authSource=admin' \
  -e ME_CONFIG_BASICAUTH_USERNAME=admin \
  -e ME_CONFIG_BASICAUTH_PASSWORD='change-this-web-password' \
  mongo-express

# Check containers and logs
docker ps
docker logs mongodb
docker logs mongo-express
```

Open Mongo Express at `http://localhost:8081` on the Docker host.

MongoDB credentials and Mongo Express web-login credentials are
separate. The Mongo Express Basic Auth settings protect its web
interface; the MongoDB URL credentials authenticate to the database.

------------------------------------------------------------------------

## 8. Docker Compose

Compose defines related services, networks, volumes, environment
settings, and dependencies in one YAML file.

### Example `compose.yaml`

This file is suitable for a local learning environment. It expects the
Node.js Dockerfile and application files in `./app/`.

``` yaml
services:
  mongodb:
    image: mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
    volumes:
      - mongo-data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo-express:
    image: mongo-express
    ports:
      - "8080:8081"
    environment:
      ME_CONFIG_MONGODB_URL: ${MONGO_URL}
      ME_CONFIG_BASICAUTH_USERNAME: ${EXPRESS_USERNAME}
      ME_CONFIG_BASICAUTH_PASSWORD: ${EXPRESS_PASSWORD}
    depends_on:
      mongodb:
        condition: service_healthy

  node-app:
    build:
      context: ./app
      dockerfile: Dockerfile
    image: node-profile-app:1.0
    ports:
      - "3000:3000"
    environment:
      MONGO_URL: ${MONGO_URL}
    depends_on:
      mongodb:
        condition: service_healthy

volumes:
  mongo-data:
```

### Example `.env.example`

``` dotenv
MONGO_USERNAME=admin
MONGO_PASSWORD=replace-with-a-strong-password
MONGO_URL=mongodb://admin:replace-with-a-strong-password@mongodb:27017/?authSource=admin
EXPRESS_USERNAME=admin
EXPRESS_PASSWORD=replace-with-a-different-strong-password
```

Copy `.env.example` to `.env`, then set matching values. The `.env` file
must be ignored by Git and must never be committed. Ensure the password
in `MONGO_URL` matches `MONGO_PASSWORD`; URL-reserved characters in
passwords must be percent-encoded.

### Compose commands

Run these commands from the directory containing `compose.yaml`:

``` bash
# Validate and render the Compose configuration
docker compose config

# Build images
docker compose build

# Build and start services in the background
docker compose up -d --build

# List services
docker compose ps

# Follow all service logs
docker compose logs -f

# Follow logs for one service
docker compose logs -f node-app

# Stop and remove containers and the default network
docker compose down

# Stop and remove containers, network, and named volumes (DESTRUCTIVE)
docker compose down -v
```

### Important Compose concepts

-   Compose creates a default network for the project unless configured
    otherwise.
-   Services discover each other using service names, such as `mongodb`.
-   `depends_on` controls startup order. `condition: service_healthy`
    waits for the configured healthcheck to pass.
-   `build` builds an image from a local Dockerfile.
-   `image` names the image to build or use.
-   A named volume preserves data beyond an individual container's
    lifecycle.
-   `docker compose up -d --build` builds and starts services; it does
    not push images to a registry.

------------------------------------------------------------------------

## 9. Node.js application and MongoDB connection

When Node.js runs **inside Compose**, connect to MongoDB using the
Compose service name:

``` text
mongodb://admin:YOUR_PASSWORD@mongodb:27017/?authSource=admin
```

When Node.js runs **directly on the Debian host**, use the published
host port:

``` text
mongodb://admin:YOUR_PASSWORD@localhost:27017/?authSource=admin
```

Prefer an environment variable in `server.js`:

``` javascript
const mongoUrl = process.env.MONGO_URL;

if (!mongoUrl) {
  throw new Error("MONGO_URL environment variable is not set");
}
```

Use `mongoUrl` in every MongoDB connection call. Do not leave an old
hardcoded `mongoUrlLocal` in one route while another route uses the
environment variable.

For example, if the app uses the MongoDB Node.js driver:

``` javascript
MongoClient.connect(mongoUrl, mongoClientOptions, function (err, client) {
  if (err) {
    console.error("MongoDB connection failed:", err);
    return;
  }

  // Continue with the application's database operation.
});
```

Adapt this example to the callback or promise style used by the actual
application and its installed driver version. Check `package.json` and
the source code before changing APIs.

**Important:** Publishing port `27017` is not required for containers on
the same Compose network. For a safer local setup, omit the MongoDB
`ports` mapping unless host access is needed.

------------------------------------------------------------------------

## 10. Publish an image to AWS ECR

Amazon Elastic Container Registry (ECR) is a managed container image
registry. The following example uses:

-   Region: `ap-southeast-2`
-   AWS CLI profile: `docker`
-   Repository: `my-app`

Replace the example account ID and repository details with your own
values.

### 10.1 Configure and verify AWS CLI

``` bash
# Configure a named profile (enter credentials when prompted)
aws configure --profile docker

# Verify the active identity for that profile
aws sts get-caller-identity --profile docker

# Check AWS CLI version
aws --version
```

Do not paste access keys into scripts, README files, shell history, or
Git repositories. Prefer IAM roles or other short-lived credentials
where available.

### 10.2 Create an ECR repository

``` bash
aws ecr create-repository \
  --repository-name my-app \
  --region ap-southeast-2 \
  --profile docker
```

If the repository already exists, inspect it rather than creating it
again:

``` bash
aws ecr describe-repositories \
  --repository-names my-app \
  --region ap-southeast-2 \
  --profile docker
```

### 10.3 Authenticate Docker to ECR

The AWS profile option belongs to the AWS CLI command, not
`docker login`:

``` bash
aws ecr get-login-password \
  --region ap-southeast-2 \
  --profile docker \
| docker login \
  --username AWS \
  --password-stdin ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com
```

Expected result:

``` text
Login Succeeded
```

If Docker is run with `sudo` but AWS credentials belong to the regular
user, the root account may not see that user's AWS profile. Prefer using
Docker consistently as one user, or deliberately configure the
credentials context securely. Do not solve this by copying secrets into
public or shared locations.

### 10.4 Build, tag, and push

``` bash
# Build the local image from the directory containing the Dockerfile
docker build -t my-app:latest .

# Tag it with the ECR repository URI
docker tag my-app:latest \
  ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/my-app:latest

# Push the image
docker push \
  ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/my-app:latest
```

### 10.5 List images and pull

``` bash
aws ecr list-images \
  --repository-name my-app \
  --region ap-southeast-2 \
  --profile docker

docker pull \
  ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/my-app:latest
```

### 10.6 Run the ECR image

``` bash
docker run -d \
  --name my-app-test \
  -p 3000:3000 \
  ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com/my-app:latest
```

This standalone command only starts the application container. If the
app requires MongoDB, it must also be attached to a network containing a
reachable MongoDB service and receive the correct `MONGO_URL`.

### 10.7 ECR IAM permissions

The identity needs permission to authenticate and upload
layers/manifests. Common actions include:

-   `ecr:GetAuthorizationToken` (typically granted on `*`)
-   `ecr:BatchCheckLayerAvailability`
-   `ecr:InitiateLayerUpload`
-   `ecr:UploadLayerPart`
-   `ecr:CompleteLayerUpload`
-   `ecr:PutImage`

Pulling images requires corresponding read permissions, such as
`ecr:BatchGetImage` and `ecr:GetDownloadUrlForLayer`.

------------------------------------------------------------------------

## 11. Troubleshooting

### Error: container name is already in use

Example:

``` text
Conflict. The container name "/mongodb" is already in use
```

Inspect existing containers:

``` bash
docker ps -a
docker inspect mongodb
```

Reuse the container, remove it if it is safe to do so, or choose another
name:

``` bash
docker stop mongodb
docker rm mongodb
```

Removing a container does not automatically remove a separately mounted
named volume.

### Error: `mongo: Name does not resolve`

Mongo Express is trying to resolve a hostname that does not exist on its
Docker network.

Check:

``` bash
docker network inspect mongo-network
docker exec mongo-express getent hosts mongodb
docker inspect mongo-express --format '{{range .Config.Env}}{{println .}}{{end}}'
```

Ensure the MongoDB container is attached to the same network and that
the configured URL uses the correct hostname. For a container named
`mongodb`, use:

``` text
mongodb://admin:PASSWORD@mongodb:27017/?authSource=admin
```

Check for a conflicting `ME_CONFIG_MONGODB_URL`; an explicit URL
pointing to `mongo` can override the intended server hostname.

### Error: Mongo Express starts but cannot authenticate

Check the MongoDB URL and `authSource=admin`. The MongoDB root user
created by `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD`
authenticates against the `admin` database.

Remember that changing the initialization environment variables on a
container with an already-initialized data volume does not reset the
existing database user's password. For a disposable lab only, a fresh
empty volume can initialize new credentials; removing a volume destroys
its data.

### Error: `docker-compose: command not found`

Use Docker Compose V2:

``` bash
docker compose version
docker compose up -d
```

The modern command has a space, not a hyphen.

### Error: `Unable to locate credentials`

Specify the AWS profile on the AWS CLI command:

``` bash
aws sts get-caller-identity --profile docker
```

For ECR login:

``` bash
aws ecr get-login-password --region ap-southeast-2 --profile docker \
| docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.ap-southeast-2.amazonaws.com
```

Do not put `--profile` after `docker login`; it is not a Docker option.

### Error: `no basic auth credentials` or ECR push denied

Reauthenticate to the exact ECR registry hostname and check that the IAM
identity has the required ECR upload permissions and that the repository
exists in the selected region/account.

### Useful diagnostics

``` bash
docker ps -a
docker images
docker network ls
docker volume ls
docker compose ps
docker compose logs --tail=100
docker system df
```

------------------------------------------------------------------------

## 12. Security notes

This is a learning lab, not a production-ready deployment. Before using
these patterns in production:

1.  Do not commit `.env`, AWS credentials, database passwords, or access
    tokens.
2.  Use strong, unique secrets and a managed secret store or Compose
    secrets where appropriate.
3.  Do not expose MongoDB (`27017`) publicly. Containers on the Compose
    network can communicate without publishing that port to the host.
4.  Restrict access to Mongo Express; do not expose its unauthenticated
    or weakly protected interface to the internet.
5.  Use IAM least privilege for ECR.
6.  Prefer short-lived AWS credentials or instance/task roles over
    long-lived access keys.
7.  Pin image versions or digests rather than relying on mutable
    `latest` tags for repeatable deployments.
8.  Run application processes as non-root where practical.
9.  Keep base images and dependencies updated; scan images for
    vulnerabilities.
10. Avoid putting passwords directly in command lines, where they may be
    retained in shell history or visible in process information.

------------------------------------------------------------------------

## 13. Practice checklist

Use this list to track hands-on progress:

-   [ ] Install Docker Engine on Debian.
-   [ ] Verify Docker and Compose versions.
-   [ ] Pull and run an Nginx container.
-   [ ] Inspect, stop, start, and remove containers.
-   [ ] Build and inspect a custom image.
-   [ ] Create a user-defined bridge network.
-   [ ] Connect MongoDB and Mongo Express using container DNS.
-   [ ] Test container-to-container connectivity.
-   [ ] Create and inspect a named volume.
-   [ ] Confirm MongoDB data persists after container recreation.
-   [ ] Write a Dockerfile for the Node.js application.
-   [ ] Configure the app using `MONGO_URL`.
-   [ ] Run MongoDB, Mongo Express, and Node.js with Compose.
-   [ ] Review Compose healthchecks and service dependencies.
-   [ ] Configure an AWS CLI profile and verify identity.
-   [ ] Create an ECR repository.
-   [ ] Authenticate Docker to ECR.
-   [ ] Tag and push an image to ECR.
-   [ ] Pull and run the ECR image.
-   [ ] Document errors and their fixes.

------------------------------------------------------------------------

## Learning outcome

By completing this lab, I practiced the core workflow of containerizing
an application, managing multi-container services, connecting an
application to a database, persisting data, and publishing a Docker
image to AWS ECR.

**Learn by doing --- Linux, Docker, DevOps, and AWS.**
