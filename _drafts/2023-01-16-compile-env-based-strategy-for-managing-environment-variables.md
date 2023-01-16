# A compile-env based strategy for managing environment variables

In this post I will describe how I deal with environment variables, and with environments in general.
The characteristics of my approach are:

- It's DRY: values are sourced from one location rather than repeated in different files
- It's modular: values are grouped by theme
- It uses secrets in a safe way
- It requires a compilation step

Before we dive into these characteristics, I would like to first address how I see the roles of the
dev, prod and - what I call - deploy environment.

## The role of the dev, prod and deploy environments.

The role of the dev environment is to help the developer to create deployable code in an efficient
and pleasant manner. This means that workflows for starting, stopping and debugging services are fast and uncomplicated,
and the system is easy to reason about so that the diagnosis of problems is made as easy as possible. The dev environment
usually contains various debugging, testing and formatting tools that help the developer.

The role of the prod environment is to execute the system in a safe, efficient and scaleable manner. Usually, developers
don't deal with this environment, only dev-ops people do.

The role of the deploy environment to "make a convincing case" that it's possible to deploy the code to production.
Ideally, when a dev-ops person looks at the deploy environment (and reads its documentation). then they can imagine how this
could be deployed to production, and they won't see any major obstacles. For example, the deploy environment should demonstate
that all required product features can be executed without having any debugging tools installed. And if the dev environment
runs a development server, then this could be replaced by a minimally configured production server in the deploy environment.
The dev-ops person can then imagine how this minimal configuration could be extended to suit the production requirements.

## The relation between the different environments

It was suggested above that the dev environment should be easy to reason about. This is important because developers
continually see their code behave in unexpected ways. To figure out what's going on, it helps if the system's environment
is relatively simple. For this reason, I don't support the idea that the dev environment should be as close as possible
to the production environment.

Now, I'm not arguing that the developer should remain completely unaware of these dev-ops related concerns. There may be
particular reasons to reflect a production concern in the dev environment. For example, in particular cases,
not including a load balancer in the dev environment may lead to a code base that does not lend itself well for load-balancing. However, since including it adds an extra burden on the developers, it should be conscious decision that is backed by analysis and arguments, not a reflex.

## Managing environment variables
