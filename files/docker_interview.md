# Docker Interview Preparation
## For DevOps Engineers (1 Year Experience)

---

## MOST ASKED INTERVIEW QUESTIONS WITH CODE

---

### Q1. What is Docker?

```
Docker packages your app and all its dependencies
into one portable unit called a container

Without Docker:
  app works on my machine
  does not work on server
  different versions of python, node etc

With Docker:
  build once
  run anywhere
  same result every time
  no more "works on my machine" problem
```

---

### Q2. Docker vs Virtual Machine

```
Virtual Machine:
  full OS inside
  takes GB of space
  slow to start (minutes)
  heavy

Docker Container:
  shares host OS kernel
  takes MB of space
  fast to start (seconds)
  lightweight
```

---

### Q3. Basic Dockerfile for Node.js app

```dockerfile
# Dockerfile

FROM node:18-alpine
# FROM = base image to start with
# node:18-alpine = node version 18 on alpine linux (small size)

WORKDIR /app
# WORKDIR = set working directory inside container
# all commands run from this directory

COPY package*.json ./
# COPY = copy files from local machine to container
# package*.json = package.json and package-lock.json
# ./ = copy to current directory (which is /app)
# copy package files first (for caching - explained below)

RUN npm install
# RUN = run command during build
# installs dependencies

COPY . .
# copy rest of code after npm install
# this is for docker layer caching
# if only code changes, npm install layer is cached

EXPOSE 3000
# EXPOSE = tell docker this app uses port 3000
# does not actually open the port, just documentation

CMD ["node", "server.js"]
# CMD = command to run when container starts
# use array format (exec form) not string (shell form)
# only one CMD per Dockerfile
```

---

### Q4. Dockerfile for Python app

```dockerfile
# Dockerfile

FROM python:3.11-slim
# python:3.11-slim = python 3.11 on debian slim (smaller than full)

WORKDIR /app

COPY requirements.txt .
# copy requirements first for caching
# if requirements dont change, pip install is cached

RUN pip install --no-cache-dir -r requirements.txt
# --no-cache-dir = dont cache pip downloads (reduces image size)

COPY . .
# copy rest of code

EXPOSE 8000

CMD ["python", "app.py"]
```

---

### Q5. Dockerfile for static HTML/CSS (nginx)

```dockerfile
# Dockerfile

FROM nginx:alpine
# nginx:alpine = nginx web server on alpine linux (very small ~5MB)

COPY . /usr/share/nginx/html
# copy html/css files to nginx default folder
# nginx serves files from this folder automatically

EXPOSE 80
# nginx listens on port 80

CMD ["nginx", "-g", "daemon off;"]
# nginx = start nginx
# -g = global config
# daemon off = run in foreground so docker knows container is running
```

---

### Q6. Multi-stage Dockerfile (important interview question)

```dockerfile
# multi-stage build = use multiple FROM statements
# final image only contains what is needed to run
# build tools are not included in final image

# Stage 1 - Build
FROM node:18 AS builder
# AS builder = name this stage "builder"

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
# this creates /app/dist folder with built files

# Stage 2 - Production
FROM nginx:alpine
# start fresh with small nginx image
# nothing from stage 1 is included automatically

COPY --from=builder /app/dist /usr/share/nginx/html
# COPY --from=builder = copy from builder stage
# copies only built files not node_modules or source code

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# Result:
# Without multi-stage: image size ~1GB (includes node, npm, source code)
# With multi-stage: image size ~50MB (only nginx + built files)
```

---

### Q7. Most used Docker commands

```bash
# images
docker build -t myapp:1.0 .          # build image from Dockerfile in current dir
docker images                         # list all images
docker pull nginx:alpine              # download image from docker hub
docker push myacr.azurecr.io/myapp   # push image to registry
docker rmi myapp:1.0                  # remove image

# containers
docker run -d -p 3000:3000 myapp:1.0  # run container
# -d = detached (background)
# -p 3000:3000 = map host port 3000 to container port 3000

docker run -d -p 3000:3000 \
  -e NODE_ENV=production \            # set environment variable
  -v /host/path:/container/path \     # mount volume
  --name mycontainer \                # give container a name
  myapp:1.0

docker ps                             # list running containers
docker ps -a                          # list all containers (including stopped)
docker stop mycontainer               # stop container
docker start mycontainer              # start stopped container
docker rm mycontainer                 # remove container
docker logs mycontainer               # view container logs
docker logs -f mycontainer            # follow logs (like tail -f)
docker exec -it mycontainer bash      # open shell inside container
docker inspect mycontainer            # detailed info about container
```

---

### Q8. Docker environment variables

```dockerfile
# method 1 - ENV in Dockerfile (baked into image)
FROM node:18-alpine
ENV NODE_ENV=production
ENV PORT=3000
ENV DB_HOST=localhost
# these are default values, can be overridden at runtime
```

```bash
# method 2 - pass at runtime
docker run -e NODE_ENV=production -e DB_HOST=mydb myapp

# method 3 - env file
docker run --env-file .env myapp

# .env file
NODE_ENV=production
DB_HOST=mydb
DB_PORT=5432
```

```yaml
# method 4 - in docker-compose.yml
services:
  app:
    image: myapp
    environment:
      - NODE_ENV=production
      - DB_HOST=mydb
    env_file:
      - .env
```

---

### Q9. Docker volumes

```bash
# volumes = persist data outside container
# when container is deleted, data in volumes is NOT deleted

# type 1 - named volume (managed by docker)
docker volume create mydata
docker run -v mydata:/app/data myapp
# data persists even if container is deleted

# type 2 - bind mount (map host folder to container)
docker run -v /home/sachin/data:/app/data myapp
# /home/sachin/data on host = /app/data in container
# good for development (code changes reflect immediately)

# type 3 - tmpfs mount (stored in memory, not on disk)
docker run --tmpfs /app/temp myapp

docker volume ls           # list volumes
docker volume rm mydata    # remove volume
docker volume prune        # remove all unused volumes
```

---

### Q10. Docker networking

```bash
# containers can communicate with each other using networks

# create network
docker network create mynetwork

# run containers on same network
docker run -d --network mynetwork --name db postgres
docker run -d --network mynetwork --name app myapp
# now app container can reach db container using hostname "db"

docker network ls              # list networks
docker network inspect mynetwork  # show network details

# port mapping
docker run -p 8080:3000 myapp
# 8080 = host port (access from outside)
# 3000 = container port (app listens here)
# open browser: localhost:8080
```

---

### Q11. Docker Compose

```yaml
# docker-compose.yml
# run multiple containers together

version: '3.8'

services:

  # web app
  app:
    build: .                    # build from Dockerfile in current dir
    ports:
      - "3000:3000"             # host:container
    environment:
      - NODE_ENV=production
      - DB_HOST=db              # use service name as hostname
    depends_on:
      - db                      # start db before app
    volumes:
      - ./app:/app              # bind mount for development
    restart: always             # restart if crashes

  # database
  db:
    image: postgres:14          # use existing image
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=secret
    volumes:
      - pgdata:/var/lib/postgresql/data   # named volume for persistence

  # nginx reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

volumes:
  pgdata:                       # declare named volume
```

```bash
docker-compose up -d            # start all services in background
docker-compose down             # stop and remove containers
docker-compose logs -f          # follow logs of all services
docker-compose ps               # list running services
docker-compose build            # rebuild images
docker-compose restart app      # restart specific service
```

---

### Q12. Docker image optimization tips

```dockerfile
# tip 1 - use alpine images (smaller size)
FROM node:18-alpine       # ~100MB
# instead of
FROM node:18              # ~900MB

# tip 2 - copy package files before source code (layer caching)
COPY package*.json ./
RUN npm install           # cached if package.json not changed
COPY . .                  # code changes dont invalidate npm install cache

# tip 3 - use .dockerignore
# .dockerignore file - files not copied into image
node_modules
.git
.env
*.log
dist
coverage

# tip 4 - combine RUN commands
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# each RUN creates a new layer
# combining reduces number of layers

# tip 5 - use multi-stage builds (shown in Q6)
```

---

### Q13. What is Docker Registry?

```
Docker Registry = place to store docker images

Public registries:
  Docker Hub         → hub.docker.com (default)
  GitHub Container   → ghcr.io

Private registries:
  Azure Container Registry (ACR)
  AWS ECR
  Google Container Registry

Commands:
  docker login myacr.azurecr.io         # login to registry
  docker tag myapp myacr.azurecr.io/myapp:1.0   # tag image
  docker push myacr.azurecr.io/myapp:1.0        # push to registry
  docker pull myacr.azurecr.io/myapp:1.0        # pull from registry
```

---

### Q14. Docker security best practices

```dockerfile
# 1. dont run as root
RUN adduser --disabled-password appuser
USER appuser          # switch to non-root user

# 2. use specific image versions (not latest)
FROM node:18.17.0-alpine   # specific version
# not FROM node:latest     # version can change unexpectedly

# 3. scan for vulnerabilities
# docker scout cves myapp
# or use trivy, snyk

# 4. dont store secrets in Dockerfile
# WRONG
ENV DB_PASSWORD=mysecret    # visible in image history

# CORRECT - pass at runtime
docker run -e DB_PASSWORD=mysecret myapp

# 5. use read-only filesystem
docker run --read-only myapp
```

---

### Common Mistakes to Avoid

```
1. Running containers as root
2. Storing secrets in Dockerfile or image
3. Using latest tag (unpredictable)
4. Not using .dockerignore
5. Not using multi-stage builds
6. Copying node_modules from host into container
7. Not understanding CMD vs ENTRYPOINT
8. Not mapping ports correctly
```

---

### CMD vs ENTRYPOINT

```dockerfile
# CMD = default command, can be overridden
CMD ["node", "server.js"]
docker run myapp              # runs node server.js
docker run myapp node other.js # overrides CMD, runs node other.js

# ENTRYPOINT = always runs, cannot be overridden
ENTRYPOINT ["node"]
CMD ["server.js"]             # default argument
docker run myapp              # runs node server.js
docker run myapp other.js     # runs node other.js (CMD is overridden but ENTRYPOINT stays)
```
