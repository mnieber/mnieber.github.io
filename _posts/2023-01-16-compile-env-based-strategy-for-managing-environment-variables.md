---
layout: post
title: 'A compile-env based strategy for managing environment variables'
date: 2023-01-16 16:21:59 +0100
categories: dev-ops
---

# A compile-env based strategy for managing environment variables

In this post I will describe how I deal with environment variables, and with environments in general.
My approach is based on the `compile-env` tool, which can be be installed with `pip install compile-env`.

The characteristics of this approach are:

- It's DRY: values are sourced from one location rather than repeated in different files
- It's modular: values are grouped by theme
- It uses secrets in a safe way
- It requires a compilation step

Before we dive into these characteristics, I would like to first address how I see the roles of the
dev, prod and - what I call - deploy environment.

## The role of the dev, prod and deploy environments

The role of the dev environment is to help the developer to create deployable code in an efficient
and pleasant manner. This means that workflows for starting, stopping and debugging services are fast and uncomplicated,
and that the system is easy to reason about so that the diagnosis of problems is made as easy as possible. For this purpose,
the dev environment usually contains various debugging, testing and formatting tools.

The role of the prod environment is to execute the system in a safe, efficient and scaleable manner. Usually, developers
don't deal with this environment, but dev-ops people do.

The role of the deploy environment to make a convincing case that it's possible to deploy the code to production.
Ideally, when a dev-ops person looks at the deploy environment (and reads its documentation) then they will be able to imagine
how this could be deployed to production. Compare this to the situation where the dev-ops engineer has to start from the
dev environment: that's far less ideal, because it burdens them with the task of first cleaning this environment from all its
dev tooling. As another example, it will typically be the case that the dev environment runs a development server, which is replaced
in the deploy environment by a minimally configured production server. This makes it easy for the dev-ops person to envision how
this minimal configuration could be extended to suit the production requirements.

## The relation between the different environments

It was suggested above that the dev environment should be easy to reason about. This is important because developers
continually see their code behave in unexpected ways. To figure out what's going on, it helps if the system's environment
is relatively simple. For this reason, I don't support the idea that the dev environment should be as close as possible
to the production environment.

Now, I'm not arguing that the developer should remain completely unaware of dev-ops related concerns. There may be
particular reasons to reflect a production concern in the dev environment. For example, in particular systems,
not including a load balancer in the dev environment may lead to a code base that does not lend itself well for load-balancing. However, since including it adds an extra burden on the developers, it should be conscious decision that is backed by analysis and arguments, not a reflex.

Note that having multiple environments is not at odds with deploying the same artifacts everywhere, and using the environment variables to tune it. These "one and only" artifacts are based on the prod environment, whereas the source code (on which prod is based) is developed inside the dev environment. There is no contradiction there.

## Security in each environment

The dev and deploy environments should have no sensitive data at all. They require some security, because we wouldn't want a hacker to gain access and see the source code. However, if a hacker obtained access, then they wouldn't be able to get any useful secrets.
Although the deploy environment doesn't have sensitive data, it will reflect some of the security measures of the prod environment. This is because the deploy environment needs to be relatively close to the prod environment (since it must make a convincing case that the system can be deployed). Finally, the prod environment obviously contains sensitive data that must be protected.

## Injecting variables into the deploy environment

In this section, I will describe how I use environment variables in my deploy environments (later, I will address the dev environment). I base this environment on docker-compose, but the same principles can be used if you are not using this tool. Inside docker-compose, environment variables are used in two ways, a dynamic and a static one:

- they can be injected into a container (dynamic).
- the container can read them from a .env file (static).

Let's first look at the dynamic case. It looks like this:

```yaml
# docker-compose.deploy.yml (The docker-compose file of the deploy environment)

version: '3.7'
services:
  backend:
    env_file: [./backend/.env/deploy.injected.env]
    image: todoapp_backend_deploy
```

As you can see, the variables are injected by reading from the `deploy.injected.env` file. Don't be confused by the fact that we are reading from a file, this is a technicality. When a dev-ops person reads `docker-compose.deploy.yml` then they will understand
that the variables in `deploy.injected.env` are injected into the container, and they will determine a mechanism to do the same in production (and probably this mechanism is _not_ based on reading a file). The `deploy.injected.env` looks like this:

```bash
# backend/.env/deploy.injected.env

# This file is automatically generated from src/env/deploy/env-spec.yaml
#
# secrets: allowed
# git    : no

DJANGO_DATABASE_PASSWORD=variety-native-enter-boat-loss
```

## Reading variables from a file in the deploy environment

We've just seen the dynamic use-case, where variables are injected into the container by docker-compose. My static use-case - where variables are read from a file - looks like this:

```bash
# backend/.env/deploy.env

# This file is automatically generated from src/env/deploy/env-spec.yaml
#
# secrets: not allowed
# git    : yes

DJANGO_SETTINGS_MODULE=app.settings.deploy
```

In the static use-case, the .env file is just another configuration file. It cannot be tweaked like the injected variables can, but it offers a central place where some of the key parameters of the system are determined. In some cases, the fact that variables are read from a file (rather than being injected) makes life easier for dev-ops people, as long as there is never any need to tune these variables later. Note that the `backend/.env/deploy.env` file documents that fact that this file may not
contain secrets and therefore can be added to git.

## Compiling .env files using compile-env

You've probably noticed that the .env files above state that they are generated automatically with compile-env. There are different reasons for generating them, rather them writing them by hand:

- the same values may appear in multiple .env files. Ideally though, we'd define these values just once.
- the .env file for a service will usually contain a mixture of variables that are related to different concerns
  (e.g. postgres, django, aws). To keep things organized, it helps to group all values related to a concern
  in a single file (e.g. `postgres.env`, `django.env`, `aws.env`), and to distribute these values to the .env files that need them.
- .env files are easier to grok if they have concrete values, rather than expressions. For example, for understanding the system
  it's more useful to see `SERVICE_VERSION=backend-2.1` in a .env file rather than `SERVICE_VERSION=${SERVICE_NAME}-${VERSION}`.
  A compilation step can take care of this type of value interpolation.

The config file for compile-env looks like this:

```yaml
# ./env/deploy/env-spec.yaml
global_dependencies: [django.env, postgres.env]

outputs:
  ../../backend/.env/deploy.injected.env:
    targets: [../../backend/.env/deploy.injected.env.in]
    dependencies: [secrets.env]
  ../../backend/.env/deploy.env:
    targets: [../../backend/.env/deploy.injected.env.in]
  ../../postgres/.env/deploy.injected.env:
    targets: ../../postgres/.env/deploy.injected.env.in]
```

To create the `.env` files that the system will use, one can simply run `cd ./env/deploy && compile-env env-spec.yaml`.
This will generate three output .env files. All of them make use of the environment variables in `django.env` and `postgres.env`.
The `deploy.injected.env` output file also makes use of `secrets.env`. One can use `git-secret` to protect this file, or keep it out of `git`
altogether.
Every output file has a related template. For `deploy.injected.env` the template is `deploy.injected.env.in` template. Templates look like normal .env files,
except that they interpolate any values that appear on the right hand (note that in the snippet below, `DJANGO_DATABASE_PASSWORD` comes from `django.env`):

```bash
# backend/.env/deploy.injected.env.in

# This file is automatically generated from src/env/deploy/env-spec.yaml
#
# secrets: allowed
# git    : no

DJANGO_DATABASE_PASSWORD=${DJANGO_DATABASE_PASSWORD}
```

## The .env files in the dev environment

The dev environment has its own `./env/dev/env-spec.yaml` file, that reads from `./env/dev/django.env`, `./env/dev/postgres.env` and
`./env/dev/secrets.env`. Since the `dev` secrets are not actually secret, you may choose to add this file to `git` (this makes it easy
for new developers to start working in this environment).

## Conclusion

I've described different environments (dev, prod and deploy) and presented a way to manage their environment variables. What I like about
this approach is that it's very structured: variables are defined (once) in a well-defined place, there are templates that determine
the shape of the .env files that the system actually uses, and the system makes it visible to the user how secrets are managed. A possible
down-side is that you have to call `compile-env` whenever you change a `compile-env` template or input file. However, these files don't
change that often. Overall, I have found the presented setup to be very enjoyable, I hope you will like it too.
