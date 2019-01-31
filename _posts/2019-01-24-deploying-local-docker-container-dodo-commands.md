---
layout: post
title:  "Deploying to a local Docker container with Dodo Commands"
date:   2019-01-24 18:42:59 +0100
categories: Docker Salt "Dodo Commands"
---
Introduction
------------

I use Docker for more or less all my local development. It allows me to test each project in an isolated environment that matches the server, with minimal dependencies on the host system. However, there are a few challenges with this approach:

- compared to running everything on the host, it requires more work to inspect the system state.
- it can be hard to integrate data which is managed by containers into the IDE. The warnings produced by Facebook's Flow type checker (which I run in Docker) are a good example.
- deploying a configuration to a Docker container is not straightforward.

The latter challenge is sometimes met by using full-blown Dockerfiles to configure the container, but this undermines the idea of using the same code everywhere: when Salt or Ansible is used on the production server then it should be used for my containers as well. This article describes an approach to achieve this. I will focus on Salt, but the case for Ansible is similar.

Note that I'm often using the terms "Docker image" and "Docker container" interchangeably. Typically, I deploy to a running container and then persist that container as an image, so in practice the difference is not so pronounced.


Global outline
--------------

A straight-forward way to deploy Salt scripts on containers is to install Salt in the container and run it locally. This has some drawbacks though: a) installing Salt bloats the container, b) it introduces a difference between how the production server and the containers are treated, and c) the Salt mechanisms for local deployment feel quite experimental compared to the default ssh-based mechanism.

For these reasons, I prefer to install Salt in a Docker image that can be used to deploy to any container in any project. Thus we are tasked with:

- installing Salt in a Docker image;
- installing the ssh-service in the target Docker image;
- running ssh-agent so that we only have to enter ssh-key passwords once;
- running ssh on the target container;
- copying our public ssh-key to `target:/root/.ssh/authorized_keys` so that Salt has ssh access;
- running Salt to deploy to the target container.

It seems like a lot of work, but most of the steps are automated with Dodo Commands. I will use the `--confirm` option in all Dodo Commands calls, so that we can still learn about the low-level calls that do the actual work.

Preparation: Dodo Commands
--------------------------

Let's install Dodo Commands and dodo_deploy_commands first.

    {% highlight bash %}
    virtualenv env
    source env/bin/activate
    pip install dodo_commands
    dodo install-default-commands --pip dodo_deploy_commands
    {% endhighlight %}


Installing Salt in a Docker image
---------------------------------

For installing Salt (and Ansible) I'm using a custom [Dockerfile](https://raw.githubusercontent.com/mnieber/dodo_deploy_commands/master/dodo_deploy_commands/drop-in/deploy-tools/docker/Dockerfile) that refers to my fork of Salt. If you want to use the master branch of Salt, please edit the url in the Dockerfile. We'll tag this image as `deploy-tools:base`.

    {% highlight bash %}
    # Get the Dockerfile
    wget https://raw.githubusercontent.com/mnieber/dodo_deploy_commands/master/dodo_deploy_commands/drop-in/deploy-tools/docker/Dockerfile ./Dockerfile.deployTools
    # Build the Dockerfile
    docker build -t deploy-tools:base -f Dockerfile.deployTools
    {% endhighlight %}


Installing ssh in the target container
--------------------------------------

I use this [Dockerfile](https://raw.githubusercontent.com/mnieber/lindyscience/master/docker/Dockerfile) as the starting point for all my projects. It installs Ubuntu and contains the following lines for setting up ssh:

    {% highlight bash %}
    RUN apt-get update && apt-get install -y openssh-server
    RUN mkdir /var/run/sshd
    RUN mkdir /root/.ssh
    EXPOSE 22
    {% endhighlight %}

Let's build this image and tag it as `target:base`:

    {% highlight bash %}
    # Get the Dockerfile
    wget https://raw.githubusercontent.com/mnieber/lindyscience/master/docker/Dockerfile ./Dockerfile.targetContainer
    # Build the Dockerfile
    docker build -t target:base -f Dockerfile.targetContainer
    {% endhighlight %}


Running ssh-agent in a container
--------------------------------

We'll install ssh-agent into a Docker image so other Docker containers can easily access it, using this [Dockerfile](https://github.com/nardeas/ssh-agent/blob/master/Dockerfile):

    {% highlight bash %}
    git clone https://github.com/nardeas/ssh-agent
    cd ssh-agent
    docker build -t ssh-agent:base
    {% endhighlight %}

We are now ready to run ssh-agent and add our key (which is named `id_rsa`):

    {% highlight bash %}
    dodo ssh-agent --confirm restart "ssh-agent:base" id_rsa

    # The calls below are automatically generated

    (/tmp) docker run -d --name=ssh-agent ssh-agent:
    confirm? [Y/n]

    (/tmp) docker run \
      --rm -it \
      --volumes-from=ssh-agent \
      -v /home/maarten/.ssh:/.ssh \
      ssh-agent:base ssh-add id_rsa
    confirm? [Y/n]

    > Enter passphrase /root/.ssh/id_rsa:
    {% endhighlight %}


The first `docker run` starts a `ssh-agent` container. The second `docker run` exits almost immediately, but it has a persistent effect on the `ssh-agent` container since we are using `--volumes-from=ssh-agent`.

Deploying to a Docker container
-------------------------------

The remaining steps are taken care of by `dodo salt-deploy`. Let's look at the code and then go through it step by step:

    {% highlight bash %}
    dodo salt-deploy --confirm \
      --ssh-key-name id_rsa \
      --verbose \
      /home/maarten/my_project/src/salt \
      target:base \
      local_docker

    # The calls below are automatically generated

    (/tmp) docker run \
      -d --rm \
      --publish=0.0.0.0:22:22 \
      --name=sshd_on_target_base \
      target:base \
      /usr/sbin/sshd -D
    confirm? [Y/n]

    (/tmp) docker cp \
      /home/maarten/.ssh/id_rsa.pub \
      sshd_on_target_base:/root/.ssh/authorized_keys
    confirm? [Y/n]

    I have created the following temporary Salt roster file:
    dev:
        host: 172.17.0.2
        priv: agent-forwarding

    I have created the following temporary Salt master file:
    file_roots:
      base:
        - /srv/salt-deploy/src/salt
    pillar_roots:
      base:
        - /srv/salt-deploy/src/salt/pillar

    (/tmp) docker run  \
      --name=salt-deploy  \
      --rm --interactive --tty  \
      --volume=/home/maarten/my_project/src/salt/.master.91298a92bfe14c5e91efc78414a4b038:/etc/salt/master  \
      --volume=/home/maarten/my_project/src/salt:/srv/salt-deploy/src/salt  \
      --volumes-from=ssh-agent  \
      --env=SSH_AUTH_SOCK=/.ssh-agent/socket  \
      --workdir=/srv/salt-deploy/src/salt  \
      deploy-tools:base  \
      salt-ssh -i --roster-file=./.roster.sshd_on_target_base local_docker state.apply
    confirm? [Y/n]

    (/tmp) docker commit sshd_on_target_base target:base
    confirm? [Y/n]

    (/tmp) docker stop sshd_on_target_base
    confirm? [Y/n]
    {% endhighlight %}

The `dodo salt-deploy` script takes the location of the Salt source files, the name of the target Docker image and the id of the server (in this case: 'local_docker') targetted by the `salt-ssh` executable. In theory, we could use any value for the server id (because we are targetting a local container, not a real server) but if the Salt script refers to specific server ids then you may want to use one of those.

Let's look at the automatically generated calls now. The first `docker run` just starts the ssh service on the target container. The call to `docker cp` copies the public ssh key. The subsequent `docker run` call is where most of the action happens:

- a temporary local `master` file (automatically generated by `dodo salt-deploy`) is mapped to location `/etc/salt/master`. This file contains the `file_roots` and `pillar_roots` for Salt (unfortunately we have to jump through hoops to tell Salt where they are). Since we used the `--verbose` option, `salt-deploy` will print this file to the screen.
- the local directory with Salt source files is mapped to location `/srv/salt-deploy/src/salt`.
- access to ssh-agent is magically obtained using `--volumes-from` and `--env`.
- `salt-ssh` is called with a temporary automatically generated `roster` file (jumping through hoops again). Note that the roster file refers to the IP address of the Docker container, and uses `priv: agent-forwarding` to allow the use of the keys managed by `ssh-agent`.

Finally, there are some calls to commit the Docker container and stop it.


Some tweaks
-----------

To keep it simple, I've used Dodo Commands without first creating a project and configuration file. In my daily work, I will store arguments such as the name of the ssh-agent image (`ssh-agent:base`) in the configuration of my project. This allows me to call `dodo ssh-agent restart` and `dodo salt-deploy` without specifying any arguments. When I want to deploy the Salt script to the production server, I call `dodo layer salt production` to overlay my configuration file with the settings stored in `layers/salt.production.yaml`. Subsequent calls to `dodo salt-deploy` will then target the server instead of the Docker container.

On the topic of sharing the source directory with the container running Salt: it's possible that `salt-ssh` needs access to not just the Salt source files but the entire source directory. This is the case when the Salt script contains a symlink to an external source file. For this reason, you can replace the `/home/maarten/my_project/src/salt` argument with `/home/maarten/my_project/src` and `--top-dir=salt`. This tells `salt-deploy` that `/home/maarten/my_project/src` must be mapped to the Docker container, but the Salt source files are found in `/home/maarten/my_project/src/salt`.

Conclusion
----------

Deploying a Salt script to a Docker container is complex, and the particulars of Salt do not make it any easier. Hopefully, the most important details to make it work have become clear in this article. Once the right Docker images are built, it's possible to capture the workflow in just two steps. Moreover, we've seen that the `--confirm` flag goes a long way in explaining what a Dodo Commands script does below the surface.

To summarize, here are the required Dodo Commands calls:

  {% highlight bash %}
  dodo ssh-agent restart "ssh-agent:base" id_rsa

  dodo salt-deploy \
    --ssh-key-name id_rsa \
    /home/maarten/my_project/src/salt \
    target:base \
    local_docker
  {% endhighlight %}

With a Dodo Commands project and the right configuration file, this reduces to:

  {% highlight bash %}
  $(dodo activate my_project)
  dodo ssh-agent restart
  dodo salt-deploy
  {% endhighlight %}
