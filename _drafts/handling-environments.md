# Handling environments

In this blog post I explain how I setup developer and production environments. Specifically, I will discuss the dev and prod environment, and what I call the 'demo' environment. For illustration purposes, I will use an example stack that contains a Django `backend` service, a React `frontend` service and a `postgres` database service.

--

## The dev environment

The dev environment aims to make life easier and better for developers.

### Environment variables

🔘 Variables are defined in `<concern-name>.env` files (in env/dev)

I try to organize the .env files in env/dev in such a way that variable definitions are easy to find. For example, django.dev will contain values such as DJANGO_DATABASE_NAME and postgres.dev will contain POSTGRES_HOST.

🔘 Secrets are defined in `secrets.env` files (in env/dev)

Every developer needs to create this file for themselves, using `secrets.env.WRITEME` as an example.

🔘 The `env-spec.yaml` file (in env/dev) maps variable definitions to each service

The `env-spec.yaml` file is used by the `compile-env` tool to create the `backend/.env/dev.env` from the `backend/.env/dev.env.in` template. This template selects values from `env/dev/django.env`, `env/dev/react.env, etc`. Since `backend/.env/dev.env` may contain secrets, it's not added to git (this is not any problem because `compile-env` generates this file).

🔘 The `<service-name>/.env/dev.env` file is self-contained.

To make the behaviour of the dev environment predictable and easy to understand, the `backend/.env/dev.env` file is self-contained. Secret values are hard-coded in this file. This means that as a developer you do not need to load or define additional variables (such as secrets) before using it. In contrast, the `backend/.env/prod.env` file usually depends on variables (i.e. secrets) that the devops engineer plugs into the environment before running the service.

### Docker-compose

🔘 The `docker-compose.dev.yml file` defines each service

🔘 Every service loads the <service-name>/.env/dev.env variables.

🔘 The `<service-name>/Dockerfile.dev` file defines the Docker image of a service.

This Dockerfile uses `CMD sleep infity`. The developer will use the Makefile of the service to perform actions.
The `postgres` service is an exception: this image has the default CMD that starts Postgres.

### Makefile

🔘 The `Makefile` contains commands for the most-used actions in a service.

For example, the `backend/Makefile` has a `run-server` command.

🔘 The `Makefile` includes `Makefile.base`

The `Makefile.base` file has commands that are shared among all environments (i.e. dev, demo and prod).

🔘 The `Makefile` assumes that all required variables are provided by the shell

This is true since the docker-compose file includes the .env file of the service.

### Python requirements.txt files

🔘 The `<service-name>/requirements/dev.in` file defines the requirements for the dev environment.

This file is compiled into the `<service-name>/requirements/dev.txt` file using the `pip-compile` tool.

🔘 The `<service-name>/requirements/dev.in` file includes `base.txt`

The `base.txt` file is created from `base.in` using pip-compile. The `base.in` file contains the requirements that are shared among all environments (i.e. dev, demo and prod).

--

## The prod environment

The prod environment runs the stack in production. This environment is typically created by dev-ops and not (directly) available to developers. However, some of the files, such as Makefile.prod and <service-name>/.env/prod.env are stored in git and therefore available to the developer.

### Environment variables

The environment variables in the prod environment are either stored in an .env file or they are injected into the container by the cloud provider. Values provided by the cloud are secrets and any value that must be configurable. It's also possible to use so-called conditional values in the .env file, using the `FOO ?= foo` syntax. In this case, `FOO` is set to the value provided by the cloud, or to `foo` if the cloud doesn't provide it.

🔘 Variables are defined in `<concern-name>.env` files (in env/prod) or in the cloud console

Similarly to dev, the variables in the `<concern-name>.env` files are compiled into `<service-name>/.env/prod.env` files by the `compile-env` tool. The `<service-name>/.env/prod.env.in` template selects the variables that are used in each service.

🔘 There is no `secrets.env` file

Instead, secrets are added in the cloud console.

🔘 The `env-spec.yaml` file (in env/prod) maps variable definitions to each service

The process is the same as for 'dev'. However, the resulting `<service-name>/.env/prod.env` file is _not_ self-contained, because some of the values come directly from the cloud. To make it explicit which variables are used in the service, you can add add a comment in the `<service-name>/.env/prod.env.in` template for every variable that is provided by the cloud.

### Kubernetes

In production, the containers are run under Kubernetes. It's also possible to use a "serverless container" feature of the cloud. Typically, a serverless container will still use Kubernetes under the surface.

🔘 The `<service-name>/Dockerfile.prod` file defines the Docker image of a service.

### Makefile

🔘 Similar to 'dev', each service has a `Makefile.prod` with commands for the most-used actions in that service.

For example, the `backend/Makefile.prod` has a `start-prod` command.

🔘 The `Makefile.prod` includes the `<service-name>/.env/prod.env` file.

By including this file, some of the required variables are provided to the rules in the makefile. The remaining required variables are provided by the cloud.

### Python requirements.txt files

🔘 Similarly to dev, the `<service-name>/requirements/prod.in` file defines the Python requirements.

This file is compiled into `<service-name>/requirements/prod.txt` by `pip-compile`.

--

## The demo environment

The demo environment is a local simulation of the prod environment. It's main purpose is to make it easier for dev-ops to create the prod environment. If a dev-ops has to use the dev environment as a starting point then they will sometimes struggle to understand which part of dev are needed in prod and which parts aren't. The stack developer can help the dev-ops by creating a demo environment that can be run locally but doesn't have any dev tooling, and that in general looks closer to the prod environment. Ideally, the dev-ops can create the prod environment from the demo environment without the need to ask too many questions.

### Environment variables

Similarly to prod, the environment variables in the demo environment are either stored in an .env file or they are injected. However, since there is no cloud, the method of injection is different. The secrets are stored in a `<service-name>/.env/demo-secrets.env` file that is injected into the container by the `docker-compose.demo.yml` file.

🔘 The `<service-name>/.env/demo-secrets.env` file is based on `<service-name>/.env/demo-secrets.env.WRITEME`

🔘 It's also possible to use a `<service-name>/.env/demo-secrets.env` file that is encrypted by git-secret

This makes it possible to share the demo secrets among developers.

🔘 The `env-spec.yaml` file (in env/demo) maps variable definitions to each service

The process is the same as for 'prod'.

### Docker-compose

Similarly to dev, the demo environment runs the stack in docker-compose. However, instead of including `<service-name>/.env/demo.env` for every service it includes `<service-name>/.env/demo-secrets.env`.

🔘 The `<service-name>/Dockerfile.demo` file defines the Docker image of a service.

This docker file will not install any dev tools. Also, it will typically be simpler than the `Dockerfile.prod` file, because it can omit various steps that harden prod or make it more efficient (e.g. caching).

### Makefile

🔘 The demo environment uses `Makefile.demo`.

This Makefile is similar to `Makefile.prod`.

### Python requirements.txt files

🔘 The demo environment uses `<service-name>/requirements/demo.in`.

This file is similar to `<service-name>/requirements/prod.in`.

--

## The local-prod environment

It's also possible to merge the demo environment into the prod environment. In this case, the `docker-compose.demo.yml` file is renamed to `docker-compose.prod.yml` and `demo-secrets.env` is renamed to `prod-secrets.env`. This option is useful when one or more production services are running as a serverless container. These serverless container typically depend on some other services provided by the cloud, such as the database, a key-value store, a message queue etc. If the local-prod environment connects to these same service, then it's able to completely manage the state of the production stack.

🔘 The local-prod environment should use the Dockerfile.prod images (or close)

There is a risk to the approach of managing the state of the production stack in two environments: these environment can have different tool-chains that use the shared resources (i.e. the database, key-value store and message queue) in subtly different ways. Normally this should be a problem because the shared resources will have an api/protocol that guarantees consistent results (regardless of how the api calls are made), but still it seems wise to minimize the gap between the toolchains of dev and local-prod. One way to do this is to use the Dockerfile.prod images in local prod, or otherwise use a Dockerfile.local-prod that installs the same tool versions.
