- Start Date: 2020-08-28
- RFC PR: [#39](https://github.com/inveniosoftware/rfcs/pull/39)
- Authors: Konstantina Stoikou, Pablo Panero

# Reproducible CI tests

## Summary

This RFC proposes a way to make test runs consistent both in Travis (or any
other CI) and a local deployment. In addition, this will provide the ways for
a test suit to easily boot up and down the required services.

## Motivation

Currently, developers have a hard time reproducing this environment as they
have to run the services locally (e.g. MySQL, PostgreSQL, several versions of
Elasticsearch), with all the management overhead that it implies. In addition,
tests do not start the required services, they assume that those are running.
This problems are represented in the following use cases:

- *As a developer I want to reproduce a specific combination of the Travis
  matrix in my local setup.*
- *As a developer I want to debug my code, with the services of a specific
  combination of the Travis matrix in my local setup (e.g. because a tests
  failed in Travis for MySQL).*
- *As a developer I want to run multiple times a single tests module/function.
  Services should not be boot up and down on every run, but be kept up.*
- *As an Invenio maintainer I want to control the service's version from a
  centralized point. e.g. ES_7_LATEST might change from 7.2.0 to 7.3.0.*
- *As an Invenio maintainer I want to see the effect of changing
  (upgrading/downgrading) a service version on the whole Invenio organization
  modules.*

## Detailed design

The creation of a new package, `docker-services-cli`, which is in charge of
managing the required services using *docker-compose*.

This makes use of a predefined `docker-services.yml`, which defines the most
common services needed to test Invenio modules and applications. However, if
needed, the CLI gives the option to provide a custom yaml file.

The cli provides two commands, `up` and `down`. These commands take a
non-limited number of services, which must exist in the services yaml file.

Example usage:

``` bash
$ docker-services-cli up es postgresql redis
$ docker-servies-cli down
```

*Note*: The `down` option does not require the services' name since
*docker-compose* keeps track of which services it booted up.

## Example

The migration requires three files changes:

- `setup.py`
- `run-tests.sh`
- `travis.yml`

**`setup.py`**

This one is easy, just add the `docker-services` dependency to the
`tests_require`.

``` python
tests_require = [
    ...
    'docker-services>=0.1.0',
    ...
]
```

**`run-tests.sh`**

In this file we have to add two lines: one at the beginning ot boot up the
services and one at the end to shut them down. For example:

```bash
docker-compose-cli up es postgresql redis
pydocstyle...
isort...
pytest
tests_exit_code=$?
docker-services-cli down
exit "$tests_exit_code"
```

In addition if there is a need to test on different service technologies, for
example `postgresql` y `mysql`, the `up` command must parametrize said
component. For example:

```bash
docker-compose services up es ${DB} redis
```

And then set an environmental variable with the value. For example:

```bash
$ export DB=postgresql
$ ./run-tests.sh
$ export DB=mysql
$ ./run-tests.sh
```

In addition, we might want to change a specific service version, by one of the
centrally configured latests or a specific one. For example:

```bash
$ export ES_VERSION=ES_7_LATEST
$ ./run-tests.sh
$ export ES_VERSION=ES_6_LATEST
$ ./run-tests.sh
$ export ES_VERSION=6.5.0
$ ./run-tests.sh
```


**`travis.yml`**

To integrate travis with the new package we have to:

1- Remove all references to external services, to avoid port collisions. For
   example:

```diff
- addons:		
-    postgresql: 9.4		
-    apt:		
-      packages:		
-        - rabbitmq-server

...

- services:
-   - mysql
-   - postgresql
-   - redis-server
```

2- Addapt the environment matrix. Note that the *docker-compose* version that
   comes with travis is old, so we have to update it:

```yaml
env:
   global:
     - DOCKER_COMPOSE_VERSION=1.26.2
   matrix:
     - REQUIREMENTS=lowest EXTRAS=all,elasticsearch5 ES_VERSION=ES_5_LATEST
     - REQUIREMENTS=lowest EXTRAS=all,elasticsearch6 ES_VERSION=ES_6_LATEST
     - REQUIREMENTS=release EXTRAS=all,elasticsearch6 ES_VERSION=ES_6_LATEST
     - REQUIREMENTS=release EXTRAS=all,elasticsearch7 ES_VERSION=ES_7_LATEST
     - REQUIREMENTS=devel EXTRAS=all,elasticsearch7 ES_VERSION=ES_7_LATEST

...

before_install:
   - sudo rm /usr/local/bin/docker-compose
   - curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-`uname -s`-`uname -m` > docker-compose
   - chmod +x docker-compose
   - sudo mv docker-compose /usr/local/bin
   - ...

...

before_script:
    - "docker --version"
    - "docker-compose --version"
```

## How we teach this

We will reference this RFC in discussion for developers. Also, the
cookiecutter module template for modules will generate the corresponding
files.

## Drawbacks

More memory might be required to run tests. However, no sign of issues has
has been seen in local tests or in Travis.

## Alternatives

Several 3rd party libraries were tested, among others `pytest-services`,
`pytest-xprocess`, `docker-services` and `pytest-docker`. These libraries
enable pytest to boot up and down services defined in `pytest.ini` or
`.services.yaml` file. They were discarded because one or more of the
following reasons:

1- We cannot use the `pytest.ini` file since it is not flexible enough
   for our configuration. We have to a `.services.yaml` approach, however
   the syntax of this file is in some cases (e.g. `docker-services`) *pure
   yaml*. This means that it is not 1-to-1 matching with `docker-compose`
   files (e.g. lists for env variables are not prefixed with `-`).
2- The mentioned services file would not be centrally managed, but come
   with each module when cookiecutted. Then, even if versions can be centrally
   managed using env variables, service configuration cannot.
3- Since it is pytest who boots up and down the services, in the case of a
   developer testing constantly a module/function (e.g. `pytest -s tests/test_example.py::test_function`)
   all services would go up and down in each run. Note that when testing
   there was no noticeable difference in time once the images where cached.
4- Some services might require liveness checkes (e.g. Elasticsearch), since
   they are slow on startup. Since this module is hooked up to pytest, it
   requires investigation on how to do so + PR and release of the library
   (This is not necesarily bad, is actually good but we have no experience
   working with the maintainer and it is a single man project).

## Unresolved questions

- What about re-downloading images, is there caching in travis?
- Will the migration be done in batch with the new
  [automation tools](https://github.com/inveniosoftware/automation-tools) or
  will it be done progresively on a per repository basis.

## Resources/Timeline

This RFC states the migration process. However, it's migration depends on the
resolved quesion above.
