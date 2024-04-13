# docker_best_practices
## Use Multi-stage Builds

## Use Multi-stage Builds

REPOSITORY   TAG                    IMAGE ID         CREATED          SIZE
python       3.12.2-bookworm        939b824ad847     40 hours ago     1.02GB
python       3.12.2-slim            24c52ee82b5c     40 hours ago     130MB
python       3.12.2-slim-bookworm   24c52ee82b5c     40 hours ago     130MB
python       3.12.2-alpine          c54b53ca8371     40 hours ago     51.8MB
python       3.12.2-alpine3.19      c54b53ca8371     40 hours ago     51.8MB

While the Alpine flavor, based on Alpine Linux, is the smallest, it can often lead to increased build times if you can't find compiled binaries that work with it. As a result, you may end up having to build the binaries yourself, which can increase the image size (depending on the required system-level dependencies) and the build times (due to having to compile from the source).

When in doubt, start with a *-slim flavor, especially in development mode, as you're building your application. You want to avoid having to continually update the Dockerfile to install necessary system-level dependencies when you add a new Python package. As you harden your application and Dockerfile(s) for production, you may want to explore using Alpine for the final image from a multi-stage build.

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