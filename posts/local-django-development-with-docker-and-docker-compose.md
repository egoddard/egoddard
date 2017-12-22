<!--
.. title: Local Django Development with Docker and docker-compose
.. slug: local-django-development-with-docker-and-docker-compose
.. date: 2017-12-19 22:21:11 UTC-06:00
.. tags: django, docker
.. category: 
.. link: 
.. description: 
.. type: text
-->

This is the repo for the December 2017 MemPy talk on using Docker Compose for Django development. We'll run through an overview of Docker, Docker Compose, and look at some basic examples. After that, we'll start setting up our Django project, starting with a basic Dockerfile and working our way up to using Docker Compose to spin up three containers: one for our app, one for Redis, and one for Postgres.

Github project: [https://github.com/egoddard/mempy-docker-django/](https://github.com/egoddard/mempy-docker-django/)

## Overview

- What is Docker?
    - Installing Docker
- Using the Docker CLI
    - __Example 1__: Hello World
    - __Example 2__: Docker processes
    - __Example 3__: SciPy up and running with the SciPy stack with one command
- Dockerizing a Django app
    - Basic Dockerfile
    - Problems with the basic config
    - Dockerfile permissions fix
    - Docker Compose Basics
    - Final Project
        - Direnv
    - Additional Resources

## What is Docker

Docker is a tool for running isolated processes or code. It is kind of like a
virtual machine, but Docker virtualizes the OS instead of the hardware. It uses
your OS's kernel space and isn't seperated from the host by the hypervisor.
This allows Docker containers to start up a lot faster (and put a lot less
strain on your battery!). Using Docker helps keep your environments consistent
across development and production.

For more info about how containers differ from virtual machines, take a look at
[What is a Container](https://www.docker.com/what-container) in the Docker
documentation.


### Installing docker

Docker can be installed from the [Docker Store](https://store.docker.com/search?type=edition&offering=community).
If you're installing on a Mac or Windows Machine, `docker-compose` is included.
If you're on linux, check out the [Docker Compose Docs](https://docs.docker.com/compose/install/)

If you're on linux, don't use the version that is in your distribution's
package manager; it is probably old. Add the official Docker repo for
your distro so you're up to date. You should also view the
[Post-Installation tips for Linux](https://docs.docker.com/engine/installation/linux/linux-postinstall/)
The commands the follow assume you have at least completed the "Manage Docker
as a non-root user" section.

## Using the Docker CLI

The following examples demonstrate creating Docker containers using the Docker
CLI. You can always run `docker` with no additional parameters to get a list
of all of the available subcommands. Similarly, running `docker <subcommand>`
with no additional parameters will provide help on that subcommand.

### __Example 1__: Hello World

Just like any programming tutorial, Docker contains a Hello World container
that you can run to make sure your Docker install is working properly:

`docker container run hello-world`

If you see the "Hello from Docker!" message, you should be ready to continue.

### __Example 2__: Docker processes 

This example shows how Docker is different from a traditional VM. The command
below starts a container named mongo (--name mongo), removes it when it is
stopped (--rm) and runs it in the background (-d):

`docker container run -d --rm --name mongo mongo`

View the running containers on your system:

`docker container ls`

Execute the command `ps aux` in the running container named mongo:

`docker container exec mongo ps aux`

We can see the same process running on the host machine:
`ps aux | grep mongod`

If we were using VirtualBox, we wouldn't be able to see the `mongod` process
running from our host container.

### __Example 3__: SciPy up and running with the SciPy stack with one command

[https://github.com/jupyter/docker-stacks/tree/master/scipy-notebook](https://github.com/jupyter/docker-stacks/tree/master/scipy-notebook)

__WARNING__: SciPy is not a small project. The container created by the
following command is several gigabytes in size.

This command runs the jupyter/scipy-notebook container interactively (-i) and
with a pseudo-TTY (-t), removes the container when it is stopped (--rm),
publishes the container's port 8888 to the host's port 8888 (-p 8888:8888), and
finally mounts a volume into the container so that files created in the container
are persistent (-v $(pwd):/home/jovyan/work).

`docker container run -it --rm -p 8888:8888 -v $(pwd):/home/jovyan/work jupyter/scipy-notebook`

Explanations for the arguments are:

* __-i__: Run the container interactively

* __-t__: Open a pseudo-TTY in the container

* __-p 8888:8888__: Publish the container's port 8888 to the host's port 8888. Since we want to be able to access the jupyter notebook server from our host machine's web browser, we have to explicitly declare the port mapping. With this option passed to `run`, we can navigate to `localhost:8888` on our host machine, and the request will be forwarded on to the container's port 8888.

* __-v $(pwd):/home/jovyan/work__: This mounts a volume from the host container into the container at the specified location. Here, we're declaring that our current working directory should be mounted in the container at `/home/jovyan/work`. With the volume mounted, any changes we make to files on our host will be accessible inside the container in the mounted directory and vice versa. Also, when the container is stopped our data will be persisted, since it was created on the host.

As you can see, there are potentially many options that can be passed to the
`docker container run` command. The 
[docker container run](https://docs.docker.com/engine/reference/commandline/container_run/)
documentation lists all of the arguments. 

In example 2, the container we run is just `mongo`, while in example 3
it is `jupyter/scipy-notebook`. The containers are fetched from
[Docker Hub](https://hub.docker.com), the official Docker container registry.
Docker Hub contains many images, some contributed by Docker (official images)
and others added by the community. You can differentiate official images from
community images based on the name: official images won't have a prefix.
Containers from other organizations or individuals will include their name as
part of the container name.

## Dockerizing a Django app

### Basic Dockerfile

So far we've been running containers from Docker Hub, unmodified. We can also
use the container images on Docker Hub as a base for creating our own
containers. To do this, we create a `Dockerfile` that specifies the Docker Hub
image to use as a base, and then we provide some additional commands. Once our
Dockerfile is ready, we can build it.

For the remaining examples to work, you'll need to clone or download the project
repo from Github:
[https://github.com/egoddard/mempy-docker-django](https://github.com/egoddard/mempy-docker-django)

In the `1-basic-dockerfile` folder is a `Dockerfile`:

```dockerfile
FROM python:3
ENV PYTHONBUFFERED 1

# App setup
RUN mkdir /app
WORKDIR /app

ADD requirements.txt /app/
RUN pip install -r requirements.txt
ADD . /app/

CMD "/bin/bash"
```

This dockerfile contains a few of the basic commands you can use in a dockerfile.

* __FROM__: Every dokerfile begins with this line. It tells Docker what image from Dockerhub to use as the base. All the commands that follow will be built on top of this image.

* __ENV__: Used to set environment variables insides the container.

* __RUN__: Run a command inside the container. This directive is usually followed by a bash command.

* __WORKDIR__: Set the working directory inside the container. Any commands that follow this will be run from this directory.

* __ADD__: Adds the source file from the container into the directory provided in the container.

* __CMD__: The default command that the container should execute when it is run. This is different than the __RUN__ directive, which is used during the build phase.

[See the Dockerfile docs](https://docs.docker.com/engine/reference/builder/)
for more in depth explanations of these and other directives that you can use
in the Dockerfile.

Docker takes the commands in a Dockerfile and builds up the container in 
layers, starting with the `FROM` directive. each directive creates a new image
that Docker uses to serve the final container. All of the layers are 'joined'
through the union file system. One of the benefits of this is that only the
parts of your container that change between builds are rebuilt, so while the
first build may take awhile (looking at you SciPy notebook), subsequent 
updates are fast. For more info about how images and layers work,
see [About images, containers, and storage drivers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/).

Build the container and then run it with the following commands:

`docker build -t django-basic .`

This command sets the tag (-t), or name, of our new image to `django-basic`.
The `.` refers to the current directory. If a Dockerfile isn't passed to the
`build command`, it will look for a `Dockerfile` in the provided directory.

`docker container run -it --rm django-basic`

You should have a shell inside the container. Try creating a Django project:

`django-admin startproject docker .`

After creating the djang project, run `ls -l` inside the container. Notice that
the files are owned by root. This is not ideal, since we don't want to have to
`chown` our files every time we create a file within the container.

In another terminal window, `cd` into the folder containing the `Dockerfile`
and run `ls`. Since we didn't map a volume (-v) when we started the container,
the files we've created so far are only inside the container. When the 
container is stopped the files will be gone.

You can also start up the development server with 
`python manage.py runserver`. However, if you try accessing the Django app
from your browser, you'll get an error because we didn't publish the ports (-p).

Use `ctrl+c` to kill the server and type `exit` to leave the container.

### Problems with the basic config

While we have a basic Django app served from a Docker container, we have some
issues to fix:

1. We can't access the server from our host machine
1. Our project disappeared when we killed the container 
1. Files created inside the container are owned by root (_may not apply_
   _to Windows and Linux environments_)

The first two issues can be corrected by passing some additional parameters in
to our `docker container run` command:

`docker container run -it -p 8000:8000 -v $(pwd):/app --rm django-basic`

This command:

  * binds ports (-p) 8000 in the container to 8000 on the host
  * creates a volume (-v) for persistent storage. Our current working directory
    on the host will be mapped to `/app` in the container.

Try creating another django project in the container and then running
`python manage.py runserver 0.0.0.0:8000`. Once the development server starts,
you should be able to visit `localhost:8000` in your browser and see the
Django 2.0 start page. When you're finished playing in the container, type
`exit` to leave the bash prompt. The container will automatically be removed
since we passed the `--rm` argument to `docker container run`.


### Dockerfile permissions fix

If you're using Docker for Windows/Mac, you may not have this problem. On
linux, we need our container user to run as a normal user, which requires
a few additions to our Dockerfile. First, we need to include 
[gosu](https://github.com/tianon/gosu) in our container, then we configure an
`ENTRYPOINT` script that sets up our non-root user. Any commands we specify in
the Dockerfile (via `CMD`) or that we pass in to the container during the
`docker container run` command will be run after the `ENTRYPOINT` script.

`cd` to the `2-dockerfile-with-entrypoint` folder. Inside is a new Dockerfile:

```dockerfile
FROM python:3
ENV PYTHONBUFFERED 1
ENV GOSU_VERSION 1.10

# Hardcode the user id to assign the non-root user for now
# We'll fix this when we get to docker-compose
ENV USER_ID 1000

# Use && to run all of these commands as a single layer
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# Install gosu so we can create a non-root user
RUN gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
    && curl -o /usr/local/bin/gosu -SL "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
    && curl -o /usr/local/bin/gosu.asc -SL "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
    && gpg --verify /usr/local/bin/gosu.asc \
    && rm /usr/local/bin/gosu.asc \
    && chmod +x /usr/local/bin/gosu

# Setup an entrypoint script to create the non-root user
COPY entrypoint.sh /usr/local/bin/entrypoint.sh

# App setup
RUN mkdir /app 
WORKDIR /app

ADD requirements.txt /app/
RUN pip install -r requirements.txt

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

In this Dockerfile, we have the new `ENTRYPOINT` directive, and we've set the
default `CMD` so that it starts the development server. The `entrypoint.sh` script:

```bash
#!/bin/bash

# Default to 1000 if the USER_ID env var isn not set
if [ -z $USER_ID ]; then
  USER_ID=1000
fi

useradd --shell /bin/bash -u $USER_ID -o -c "" -m user
export HOME=/home/user

# Docker created the /app dir as root, so we need to chown it
chown -R user:user /app

# Run the gosu command as our new user. Any commands sent to
# the container (from the CMD directive or via CLI) will be 
# filled in at the "$@" and run as the non-root user
exec /usr/local/bin/gosu user "$@"
```

Build the container and then run `bash` in the container, overriding the default command:

`docker build -t django-permissions .`

`docker container run -it --rm -p 8000:8000 -v $(pwd):/app django-permissions bash`

You should have a shell inside the container. Try creating a Django project:

`django-admin startproject docker .`

Now type `exit` to leave the container. If you run `ls -l` in your host's
terminal, the django files should be owned by your host user. Now run the
`django-permissions` container with no command. The development server should
start:

`docker container run --rm -p 8000:8000: -v $(pwd):/app django-permissions`

Visiting `localhost:8000` in your host's browser should show the Django 2.0
start page. It works! That's a lot of command to run and parameters to 
remember everytime you want to start your development environment though. we
can simplify this setup and make it more reproducible with Docker Compose.

### Docker Compose Basics

Docker compose is a tool for defining and running multiple containers. It is
used primarily for local development and CI workflows, however recent versions
can also be used in production with Docker Swarm. Just like Docker and the
`Dockerfile`, `docker-compose` allows you to create a `docker-compose.yml`
file that defines a number of services (containers) and their configuration.

`cd` to the `3-docker-compose` folder. We still have our `Dockerfile`,
`entrypoint.sh`, and `requirements.txt`, but there is also a new file:
`docker-compose.yml`. We use this file to define the containers we want to
start, set their configuration options such as volumes, exposed ports, and
environment variables, and also set up the relationships between containers:

```yaml
version: '3'

services:
  postgres:
    image: postgres:9.6
    ports:
      - "5432:5432"
    environment:
      PGDATA: '/var/lib/postgresql/data/pgdata'
      POSTGRES_DB: 'my_django_app'
      POSTGRES_USER: 'my_django_app'
      POSTGRES_PASSWORD: 'mysecretpassword'
    volumes: 
      - django-data:/var/lib/postgresql/data/pgdata

  django:
    image: docker-django
    build: .
    working_dir: /app
    command: python manage.py runserver 0.0.0.0:8000
    env_file:
      - django_env
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - postgres

volumes:
  django-data:
```

Each value under the `services` key is a container. This compose file sets up
two containers: `postgres` and `django`. The `postgres` container pulls the
official `postgres` image from Dockerhub. We publish container port 5432 to
host port 5432, just in case we want to use `psql` to inspect our database.
Then we configure environment variables and create a volume so that the
database files are stored on our host for persistence. If the `django-data`
volume doesn't exist, `docker-compose` will create it.

The `django` container is a little more interesting. We configure
`docker-compose` to build an image for the `django` container named
`docker-django` using the Dockerfile in the current directory (`build: .`).
The default command can be set here. It can be different from the one set
in the `Dockerfile`. Instead of listing the environment variables directly, we
can point `docker-compose` at a file containing all the environment variables
that we need in our container. We also have a volume setup, but this time we're
mapping the current directory into the `/app` folder. This will allow our
container to see changes in real time so it can restart the development server
whenever a change is detected. Finally, we publish port 8000 to the host so that
we can connect to the app in our browser, and then we let Docker know that that
the `django` container depends on `postgres`.

You can see that there are several parameters that map to some of the options
we passed to `docker container run`. With the `docker-compose.yml` file
configured, running `docker-compuse up` starts up both our `django` (serving code
from the current directory) and `postgres` database containers defined in the
compose file.

Just like with Docker, we need to build the container images before we run them
with `docker-compose`. To build all of the containers defined in the compose
file, execute:

`docker-compose build`

Anytime you make changes to the Dockerfile or other files that the container
depends on, such as `entrypoint.sh` or `requirements.txt`, you should re-run
`docker-compose build`. Creating new files for Django or running migrations
does __not__ require a rebuild, since those file's aren't used in the
container's build phase.

After building the container, start them up:

`docker-compose up`

You should see the logs of the postgres container getting started, and you may
also see an error about `manage.py` not being found if you haven't created a
Django project yet. Before we do that, lets start a `bash` prompt and check
out container networking.

Arbitrary commands can be run in the container with
`docker-compose run <service> <cmd>`:

`docker-compose run django bash`

`docker-compose` automatically creates a network for the services listed in a
`docker-compose.yml` file. The network is configured to use the service name
for DNS. From the bash prompt in the `django` container, we can ping the
postgres container using `ping postgres`.

Use `exit` to leave the container bash prompt, followed by
`docker-compose down` to shut down all of the services specified in the compose
file.

We can also use the `docker-compose run` command to execute one-off commands,
such as starting a Django project:

`docker-compose run django django-admin startproject docker .`

Try starting the containers again, this time using just `docker-compose run django`.

The development server should be running, but navigating `localhost:8000`
throws an error. Even though we've mapped the ports in `docker-compose.yml`,
the `docker-compose run` command ignores port mappings so that it doesn't
conflict with `docker-compose up`. If the ports are already mapped, bringing
the containers up will fail. You can pass the `--service-ports` parameter in to
`docker-compose run` to explicitly map the ports:

`docker-compose run --service-ports django`

Go ahead and use `docker-compose down` to stop all of the containers, and then run `docker-compose up`. you should see the postgres and django logs, and be able to access the Django app at `localhost:8000`.

If you can do the same thing with `docker-compose up`
and docker-compose run --service-ports`, why are there two commands, and when do
you use each? Use `docker-compose run --service-ports` when you need to do
interactive debugging. For example, if you use `ipdb` and you add an
`ipdb.set_trace()` somewhere in your code, when that breakpoint is hit you will
not be able to access an interactive ipython shell if you launched your containers
with `docker-compose up`. It is not designed for interactivity. Use
`docker-compose run --service-ports` instead.

We can take advantage of this setup and some environment variables to use
`docker-compose` to closely mimic our production environment. For our final
project we'll setup a Django Rest Framework API to serve some spatial data
using `docker-compose`.

### Final Project
The app in this project is a dockerized version of
https://github.com/egoddard/mempy-drf-gis. For more info about that project,
visit the repo. You should be able to follow along in that repo if you want
more info about that project, just skip any steps that mention `vagrant`,
`pip`, `mkvirtualenv` or steps involving database creation; all of that is
handled by Docker. Unlike the other examples so far, this example has a
complete Django project that we'll walk through to see how it is configured for
Docker.

#### [Direnv](https://direnv.net/)

In the last example's `docker-compose.yml`, you may have noticed we had
passwords and database urls, etc. in our compose file, which isn't good because
we want to commit that to version control. A better option is to load those
values from the environment. We can use `direnv` to automatically load/unload
environment variables when we enter directories, which we can reference in our
`docker-compose.yml`

__NOTE__: I am not sure if this direnv/docker setup works with Windows. I'm
pretty sure it works with Mac, and I know it works in Linux. If you get errors,
you may have to change the following docker-compose file to use the `env:` as
in the `django` container in the previous docker-compose example.

To make sure your environment variables are never committed to version control,
create a global `gitignore` file that contains `.envrc`, the file `direnv`
looks for in a directory:

```bash
touch ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global
echo ".envrc" >> ~/.gitignore_global
```

After creating the ~/.gitignore_global file (if you so choose), `cd` into the
`4-final-project` folder. Create a `.envrc` file with the following content:

```bash
export POSTGRES_DB=osm
export POSTGRES_USER=osm
export POSTGRES_PASSWORD=mysecretpassword
export DATABASE_URL=postgis://osm:mysecretpassword@postgres:5432/osm
export SECRET_KEY='super_secret_key'
export DEBUG=True
```
After creating a `.envrc` file in a directory, any time you make changes to the
file you need to run `direnv allow` while in the directory before the file will
be loaded into your environment.

We can reference these variables directly in our

```yaml
version: '3'

services:
  postgres:
    image: mdillon/postgis:9.6-alpine
    ports:
      - "5432:5432"
    environment:
      PGDATA: '/var/lib/postgresql/data/pgdata'
      POSTGRES_DB: $POSTGRES_DB
      POSTGRES_USER: $POSTGRES_USER
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
    volumes: 
      - django-data:/var/lib/postgresql/data/pgdata

  django:
    image: docker-django
    build: .
    working_dir: /app
    command: python manage.py runserver 0.0.0.0:8000
    environment:
      DATABASE_URL: $DATABASE_URL
      SECRET_KEY: $SECRET_KEY
      DEBUG: $DEBUG
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - postgres

volumes:
  django-data:
```

This project's `docker-compose.yml` file is basically the same as our previous
version, but I've changed the image used to build the `postgres` container.
Since we're working with spatial data, we want our postgres database to include
the [PostGIS](http://www.postgis.net) extension. PostGIS adds spatial data
column types and includes many spatial functions we can take advantage of, such 
as finding distances between locations, finding all the points within a
boundary, calculating intersections of features, and many more.

The official `postgres` container doesn't have postgis, but there are other
containers that build on the official container to add extra features. We'll
use one of these to get postgis support.

With the variables in the container, we can update
`settings.py` to use
[djang-environ](https://github.com/joke2k/django-environ) to configure our 
database, secret key, and debug settings from environment varibales.

You can view the full `settings.py` in the Github project, but the changes we
need to make involve importing `environ` and populating some of the required
configuration constants with values from our environment:

```python
import os
import environ

# Build paths inside the project like this: os.path.join(BASE_DIR, ...)
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
root = environ.Path(BASE_DIR)

env = environ.Env()

SITE_ROOT = root()

ALLOWED_HOSTS = []
# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env('DEBUG')
TEMPLATE_DEBUG = DEBUG

SECRET_KEY = env('SECRET_KEY')

DATABASES = {
    'default': env.db(),
}
```

Build the containers with `docker-compose build`, and then run them with
`docker-compose up`. The Django development server will automatically start.

In another terminal, run
`docker-compose run django python manage.py makemigrations osm` to
generate the migration for the `osm` app. The `osm` app contains a model,
view, and serializer. They contain standard
[Django Rest Framework](http://www.django-rest-framework.org/) features,
except the serializer. The serializer also uses [DRF-GIS](https://github.com/djangonauts/django-rest-framework-gis).
DRF-GIS, like PostGIS, adds spatial serializers and methods to Django Rest
Framework. Remember, you can read more about the
[example project](https://github.com/egoddard/mempy-drf-gis) if you're
interested.

Run `docker-compose run django python manage.py migrate` to apply all migrations.

Finally, to load the data from the `osm_amenities.json` fixtures, run:

`docker-compose run django python manage.py loaddata osm_amenities.json`

Visit `localhost:8000` in your browser to see the browsable API.

### Additional Resources

* [Getting started with Docker](https://docs.docker.com/get-started/)
* [Play with Docker](https://labs.play-with-docker.com/)
* [Quickstart with Compose and Django](https://docs.docker.com/compose/django/)
* [Dockerfile Documentation](https://docs.docker.com/engine/reference/builder/)
* [Compose Documentation](https://docs.docker.com/compose/)
* [Django Rest Framework](http://django-rest-framework.org/)
* [DRF-GIS](https://github.com/djangonauts/django-rest-framework-gis)