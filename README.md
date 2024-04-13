# ff

pip3 freeze > requirements.txt

# docker_best_practices

https://testdriven.io/blog/docker-best-practices/

docker image build -t flask_docker .

docker images

docker run -p 5000:5000 -d/it flask_docker

You can list Docker containers:

$ docker ps
And you can inspect a container:

$ docker inspect
You can view Docker logs in a Docker container:

$ docker logs
And you can stop a running container:

$ docker stop

## Use Multi-stage Builds

Multi-stage Docker builds allow you to break up your Dockerfiles into several stages. For example, you can have a stage for compiling and building your application, which can then be copied to subsequent stages. Since only the final stage is used to create the image, the dependencies and tools associated with building your application are discarded, leaving a lean and modular production-ready image.

Size comparison:

REPOSITORY                  TAG                   IMAGE ID       CREATED         SIZE

ds-multi                    latest                b4195deac742   2 minutes ago   357MB

ds-single                   latest                7c23c43aeda6   6 minutes ago   969MB

In summary, multi-stage builds can decrease the size of your production images, helping you save time and money. In addition, this will simplify your production containers. Also, due to the smaller size and simplicity, there's potentially a smaller attack surface.

# Order Dockerfile Commands Appropriately

Pay close attention to the order of your Dockerfile commands to leverage layer caching.

Docker caches each step (or layer) in a particular Dockerfile to speed up subsequent builds. When a step changes, the cache will be invalidated not only for that particular step but all succeeding steps.

Example:

FROM python:3.12.2-slim

WORKDIR /app

COPY sample.py .

COPY requirements.txt .

RUN pip install -r requirements.txt

In this Dockerfile, we copied over the application code before installing the requirements. Now, each time we change sample.py, the build will reinstall the packages. This is very inefficient, especially when using a Docker container as a development environment. Therefore, it's crucial to keep the files that frequently change towards the end of the Dockerfile.

You can also help prevent unwanted cache invalidations by using a .dockerignore file to exclude unnecessary files from being added to the Docker build context and the final image. More on this here shortly.

So, in the above Dockerfile, you should move the COPY sample.py . command to the bottom:

FROM python:3.12.2-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY sample.py .

Notes:

Always put layers that are likely to change as low as possible in the Dockerfile.

Combine RUN apt-get update and RUN apt-get install commands. (This also helps to reduce the image size. We'll touch on this shortly.)

If you want to turn off caching for a particular Docker build, add the --no-cache=True flag.


## Use Smaill Docker Base Images

REPOSITORY   TAG                    IMAGE ID         CREATED          SIZE

python       3.12.2-bookworm        939b824ad847     40 hours ago     1.02GB

python       3.12.2-slim            24c52ee82b5c     40 hours ago     130MB

python       3.12.2-slim-bookworm   24c52ee82b5c     40 hours ago     130MB

python       3.12.2-alpine          c54b53ca8371     40 hours ago     51.8MB

python       3.12.2-alpine3.19      c54b53ca8371     40 hours ago     51.8MB


While the Alpine flavor, based on Alpine Linux, is the smallest, it can often lead to increased build times if you can't find compiled binaries that work with it. As a result, you may end up having to build the binaries yourself, which can increase the image size (depending on the required system-level dependencies) and the build times (due to having to compile from the source).

When in doubt, start with a *-slim flavor, especially in development mode, as you're building your application. You want to avoid having to continually update the Dockerfile to install necessary system-level dependencies when you add a new Python package. As you harden your application and Dockerfile(s) for production, you may want to explore using Alpine for the final image from a multi-stage build.



## Minimize the Number of Layers

It's a good idea to combine the RUN, COPY, and ADD commands as much as possible since they create layers. Each layer increases the size of the image since they are cached. Therefore, as the number of layers increases, the size also increases.

RUN apt-get update
RUN apt-get install -y netcat
Can be combined into a single RUN command:

RUN apt-get update && apt-get install -y netcat

Notes:

RUN, COPY, and ADD each create layers.
Each layer contains the differences from the previous layer.
Layers increase the size of the final image.
Tips:

Combine related commands.
Remove unnecessary files in the same RUN step that created them.
Minimize the number of times apt-get upgrade is run since it upgrades all packages to the latest version.
With multi-stage builds, don't worry too much about overly optimizing the commands in temp stages.
Finally, for readability, it's a good idea to sort multi-line arguments alphanumerically:

RUN apt-get update && apt-get install -y \
    git \
    gcc \
    matplotlib \
    pillow  \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

For example, after installing packages with apt-get, use && apt-get clean && rm -rf /var/lib/apt/lists/* to remove the package lists and any temporary files created during the installation process, as demonstrated above. This practice is essential for keeping your Docker images as lean and efficient as possible.

## Use Unprivileged Containers

By default, Docker runs container processes as root inside of a container. However, this is a bad practice since a process running as root inside the container is running as root in the Docker host. Thus, if an attacker gains access to your container, they have access to all the root privileges and can perform several attacks against the Docker host, like-

copying sensitive info from the host's filesystem to the container
executing remote commands
To prevent this, make sure to run container processes with a non-root user:

RUN addgroup --system app && adduser --system --group app

USER app
You can take it a step further and remove shell access and ensure there's no home directory as well:

RUN addgroup --gid 1001 --system app && \
    adduser --no-create-home --shell /bin/false --disabled-password --uid 1001 --system --group app

USER app
Verify:

docker run -i sample id

uid=1001(app) gid=1001(app) groups=1001(app)
Here, the application within the container runs under a non-root user. However, keep in mind, the Docker daemon and the container itself still run with root privileges. Be sure to review Run the Docker daemon as a non-root user for help with running both the daemon and containers as a non-root user.

## Prefer COPY Over ADD

Use COPY unless you're sure you need the additional functionality that comes with ADD.

What's the difference between COPY and ADD?

Both commands allow you to copy files from a specific location into a Docker image:

ADD <src> <dest>
COPY <src> <dest>
While they look like they serve the same purpose, ADD has some additional functionality:

COPY is used for copying local files or directories from the Docker host to the image.
ADD can be used for the same thing as well as downloading external files. Also, if you use a compressed file (tar, gzip, bzip2, etc.) as the <src> parameter, ADD will automatically unpack the contents to the given location. copy local files on the host to the destination

COPY /source/path  /destination/path
ADD /source/path  /destination/path

 download external file and copy to the destination

ADD http://external.file/url  /destination/path

 copy and extract local compresses files

ADD source.file.tar.gz /destination/path

## Cache Python Packages to the Docker Host

When a requirements file is changed, the image needs to be rebuilt to install the new packages. The earlier steps will be cached, as mentioned in Minimize the Number of Layers. Downloading all packages while rebuilding the image can cause a lot of network activity and takes a lot of time. Each rebuild takes up the same amount of time for downloading common packages across builds.

You can avoid this by mapping the pip cache directory to a directory on the host machine. So for each rebuild, the cached versions persist and can improve the build speed.

Add a volume to the docker run as -v $HOME/.cache/pip-docker/:/root/.cache/pip or as a mapping in the Docker Compose file.

The directory presented above is only for reference. Make sure you map the cache directory and not the site-packages (where the built packages reside).

Moving the cache from the docker image to the host can save you space in the final image.

If you're leveraging Docker BuildKit, use BuildKit cache mounts to manage the cache:

 syntax = docker/dockerfile:1.2

...

COPY requirements.txt .

RUN --mount=type=cache,target=/root/.cache/pip \
        pip install -r requirements.txt
    
## Run Only One Process Per Container

Why is it recommended to run only one process per container?

Let's assume your application stack consists of a two web servers and a database. While you could easily run all three from a single container, you should run each in a separate container to make it easier to reuse and scale each of the individual services.

Scaling - With each service in a separate container, you can scale one of your web servers horizontally as needed to handle more traffic.

Reusability - Perhaps you have another service that needs a containerized database. You can simply reuse the same database container without bringing two unnecessary services along with it.
Logging - Coupling containers makes logging much more complex. We'll address this in further detail later in this article.

Portability and Predictability - It's much easier to make security patches or debug an issue when there's less surface area to work with.

## Prefer Array Over String Syntax

You can write the CMD and ENTRYPOINT commands in your Dockerfiles in both array (exec) or string (shell) formats:

- array (exec)

CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "main:app"]

- string (shell)

CMD "gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app"

Both are correct and achieve nearly the same thing; however, you should use the exec format whenever possible. From the Docker documentation:

Make sure you're using the exec form of CMD and ENTRYPOINT in your Dockerfile.

For example use ["program", "arg1", "arg2"] not "program arg1 arg2". Using the string form causes Docker to run your process using bash, which doesn't handle signals properly. Compose always uses the JSON form, so don't worry if you override the command or entrypoint in your Compose file.

So, since most shells don't process signals to child processes, if you use the shell format, CTRL-C (which generates a SIGTERM) may not stop a child process.

Example:

FROM ubuntu:24.04

- BAD: shell format

ENTRYPOINT top -d

- GOOD: exec format

ENTRYPOINT ["top", "-d"]
Try both of these. Take note that with the shell format flavor, CTRL-C won't kill the process. Instead, you'll see ^C^C^C^C^C^C^C^C^C^C^C.

Another caveat is that the shell format carries the PID of the shell, not the process itself.

- array format

root@18d8fd3fd4d2:/app# ps ax

  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 python manage.py runserver 0.0.0.0:8000
    7 ?        Sl     0:02 /usr/local/bin/python manage.py runserver 0.0.0.0:8000
   25 pts/0    Ss     0:00 bash
  356 pts/0    R+     0:00 ps ax


-  string format

root@ede24a5ef536:/app# ps ax

  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 /bin/sh -c python manage.py runserver 0.0.0.0:8000
    8 ?        S      0:00 python manage.py runserver 0.0.0.0:8000
    9 ?        Sl     0:01 /usr/local/bin/python manage.py runserver 0.0.0.0:8000
   13 pts/0    Ss     0:00 bash
  342 pts/0    R+     0:00 ps ax

## Include a HEALTHCHECK Instruction

Use a HEALTHCHECK to determine if the process running in the container is not only up and running, but is "healthy" as well.

Docker exposes an API for checking the status of the process running in the container, which provides much more information than just whether the process is "running" or not since "running" covers "it is up and working", "still launching", and even "stuck in some infinite loop error state". You can interact with this API via the HEALTHCHECK instruction.

For example, if you're serving up a web app, you can use the following to determine if the / endpoint is up and can handle serving requests:

HEALTHCHECK CMD curl --fail http://localhost:8000 || exit 1
If you run docker ps, you can see the status of the HEALTHCHECK.

Healthy example:

CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                            PORTS                                       NAMES
09c2eb4970d4   healthcheck   "python manage.py ru…"   10 seconds ago   Up 8 seconds (health: starting)   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp   xenodochial_clarke
Unhealthy example:

CONTAINER ID   IMAGE         COMMAND                  CREATED              STATUS                          PORTS                                       NAMES
09c2eb4970d4   healthcheck   "python manage.py ru…"   About a minute ago   Up About a minute (unhealthy)   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp   xenodochial_clarke
You can take it a step further and set up a custom endpoint used only for health checks and then configure the HEALTHCHECK to test against the returned data. For example, if the endpoint returns a JSON response of {"ping": "pong"}, you can instruct the HEALTHCHECK to validate the response body.

Here's how you view the status of the health check status using docker inspect:

❯ docker inspect --format "{{json .State.Health }}" ab94f2ac7889
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2021-09-28T15:22:57.5764644Z",
      "End": "2021-09-28T15:22:57.7825527Z",
      "ExitCode": 0,
      "Output": "..."
Here, the output is trimmed as it contains the whole HTML output.

You can also add a health check to a Docker Compose file:

version: "3.8"

services:
  web:
    build: .
    ports:
      - '8000:8000'
    healthcheck:
      test: curl --fail http://localhost:8000 || exit 1
      interval: 10s
      timeout: 10s
      start_period: 10s
      retries: 3
Options:

test: The command to test.
interval: The interval to test for -- e.g., test every x unit of time.
timeout: Max time to wait for the response.
start_period: When to start the health check. It can be used when additional tasks are performed before the containers are ready, like running migrations.
retries: Maximum retries before designating a test as failed.
If you're using an orchestration tool other than Docker Swarm -- e.g., Kubernetes or AWS ECS -- it's highly likely that the tool has its own internal system for handling health checks. Refer to the docs of the particular tool before adding the HEALTHCHECK instruction.

## Prefer Array Over String Syntax

Use a .dockerignore File

We've mentioned using a .dockerignore file a few times already. This file is used to specify the files and folders that you don't want to be added to the initial build context sent to the Docker daemon, which will then build your image. Put another way, you can use it to define the build context that you need.

When a Docker image is built, the entire Docker context -- e.g., the root of your project -- is sent to the Docker daemon before the COPY or ADD commands are evaluated. This can be pretty expensive, especially if you have many dependencies, large data files, or build artifacts in your project. Plus, the Docker CLI and daemon may not be on the same machine. So, if the daemon is executed on a remote machine, you should be even more mindful of the size of the build context.

What should you add to the .dockerignore file?

Temporary files and folders
Build logs
Local secrets
Local development files like docker-compose.yml
Version control folders like ".git", ".hg", and ".svn"
Example:

**/.git
**/.gitignore
**/.vscode
**/coverage
**/.env
**/.aws
**/.ssh
Dockerfile
README.md
docker-compose.yml
**/.DS_Store
**/venv
**/env
In summary, a properly structured .dockerignore can help:

Decrease the size of the Docker image
Speed up the build process
Prevent unnecessary cache invalidation
Prevent leaking secrets