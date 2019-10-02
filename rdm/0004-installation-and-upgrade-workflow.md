- Start Date: 2019-10-01
- RFC PR: [#4](https://github.com/inveniosoftware/rfcs/pull/4)
- Authors: Pablo Panero

# Installation and upgrade workflow

## Summary

Overall design of the installation and upgrade workflows of Invenio RDM.

## Motivation

- As a developer, I want to be able to initialize the RDM file/folder structure, so that I can have a working instance.
- As a developer, I want to lock the dependencies of my application, so that I can control them through a VCS (Version Control System) system.
- As a developer, I want to be able to build a container image with the dependencies of my application, so I do not have to install them everytime I build the application image.
- As a developer, I want to be able to build a container image of my application.
- As a developer, I want to be able to run the application and the services it depends on as containers.
- As a developer, I want to be able to run the application in a development server, so that I can easily debug and profile my application.
- As a developer, I want to be able to stop a running instance and its corresponding services without loosing the data that is alocated in them.
- As a developer, I want to be able to clean up the environment and all the services, so that I can start from scratch.
- As a developer, I want to be able to update my running instance, so that I can apply changes such as customizations.
- As a developer, I want to be able to upgrade my running instance to a newer versions of Invenio RDM.

## Detailed design

### CLI and commands
Invenio RDM installation and upgrade procedures can be carried out using a CLI (Command Line Interface). This CLI provides the following commands:

- `init`
- `build`
- `setup`
- `run`
- `update`
- `upgrade`
- `destroy`

**init**

The `init` command would initialize the files and folders structure, resulting in a working instance. The initialization will be performed by prompting the user with a set of questions that allow to customize the system architecture (i.e. project name, Elasticsearch version, relational database system). This boostraping is implemented as a Cookiecutter module.

The resulting file structure is as follows:

- `invenio.cfg`: Contains all the configuration variables.
- `pipenv`: Contains the python dependencies along with each corresponding version.
- `static/`: Contains all the CSS and potential static content (e.g. images).
- `templates/`: Contains the custom templates.
- `Dockerfile.base`: Description of the process to install the Python dependencies.
- `Dockerfile`: Description of the process to create the application image.
- `docker-services.yml`: Contains the description of the containers for all needed services.
- `docker-compose.yml`: Contains the service components architecture and orchestration for a semi-production deployment. It uses the services described in the `docker-services.yml` file.
- `docker-compose.dev.yml`: Contains the service components architecture and orchestration for a development deployment. It uses the services described in the `docker-services.yml` file.

**build**

The `build` command should allow to build two types of environments: Development and semi-production.

Nonetheless, independently from the environment type, the first step of the build process is to lock the application dependencies. Therefore, by using Pipenv, the CLI will generate a *lock* file that would contain the version and hash identifier of each dependency. For this use case the flags ``--lock`` or ``--no-lock`` are used, being the first the default one.

The *development* environment would run the application using Flask's debug server and connect to the cache, database, queueing and elasticsearch services running in their corresponding containers. In this case both UI and REST applications would run in the same application.

The *production* environment is more complex. It runs the web UI and RESTful API in separate containers, and exposes them by using a Nginx frontend. In this scenario both the base and application images must be built. However, in cases such as updating the application the dependencies might not change, and therefore there is no need to rebuild the base image. To make this possible the flags `--base` and `--prod` are used.

Finally to be able to specify which environment should be built, the CLI offers the `--dev` flag.

Note that the `build` command creates (but does not start) the containers.

**setup**

The `setup` command is required to be run only once, upon instance creation. This command sets the environment:

- Destroy the database if existing.
- Create the database and initialize it with the needed tables.
- Destroy indeces if existing.
- Create the indeces.
- Purge the queue.
- Create a default location for files.
- Create an adming role in order to restrict access.

**run**

The `run` command boots up the whole system. It starts the services containers, and depending on the environment the application containers (*production*) or debug server (*development*).

**update**
*Needed? See unresolved questions*
The `update` command allows users to update there instances, in terms of configuration or customizations. This would require the rebuild of the application image.

**upgrade**

The `upgrade` command allows users to upgrade there instances, in terms of Invenio RDM versions. This will require the rebuild of the application image, and potentially the base image. Since this might also carry mayor canges, e.g. in the datamodel, the Elasticsearch and Database services would need to be updated.

**cleanup**
*Needed? See unresolved questions*

**destroy**

The `destroy` command results in the removal of all data, volumnes, containers and application files generated by the `build` and `run` command. Nonetheless, the file structure and contents generated by `init` will remain.

## Usage

In order to use the CLI, it needs to be installed. It is distributed as a package and can be installed via `pip`.

``` bash
$ pip install invenio-cli
```

This package will install Cookiecutter, PyDocker and Pipenv among others.

It has a series of hard dependencies that should be installed by the user:

- Python 3.4+
- Docker
- Docker Compose v2.x

An example execution to deploy a development instance would be:

``` bash
$ invenio-cli rdm init
Initializing files and folders for RDM repository...

$ invenio-cli rdm build --lock --dev
Locking dependencies...
Creating services\' containers...

$ invenio-cli rdm run --dev
Starting services\' containers...
Starting application server (Debug mode)...

$ invenio-cli rdm setup --dev
Creating database tables...
Creating indeces and templates...
Creating queues on message broker...
Creating files location...
Loading initial users, roles and permissions...

$ invenio-cli rdm destroy
Stopping application...
Stopping services...
Removing containers...
Removing volumes...
Success! The application was destroyed!
```

## How we teach this

Installation and upgrading workflows and a big change from the currently used ones in the Invenio Framework. Step by step tutorials in the Invenio RDM documentation might be created.

To help the adoption of this tool, a link in the main webpage can be set up.

## Drawbacks

Users might not easily find the difference between this and the per-module already existing CLI. Therefore, this CLI is called `invenio-cli`.

## Unresolved questions

- Is the differentiation between `update`/`upgrade` needed? Or it should be considered as one (e.g. with different flags).
- `destroy` would destroy the full containers, volumes, etc. and bring the system to a state where the developer can start from scratch. Is there a need for `cleanup` command that does not destroy the services but cleans up database tables and elasticsearch indeces? This could be achieved by calling `setup`.
- Is the division between base and application image needed? Does docker build caching system solve the problem we are trying to address?
- Command examples might vary in the future (e.g. CLI name might change). Maybe is better to be put in the module documentation. To allow deprecation, etc.
- Shall we create an RFC for CLI and then a specific RFC for each flavour that can be installed with the corresponding CLI?
- "semi-production" is used along the RFC, shall it be changed by "pseudo". The word production might give the impresion that it can be run as is. However, there are many other things to take into account in a production deployment (replication, persistency, HA of each component, security, etc.)
