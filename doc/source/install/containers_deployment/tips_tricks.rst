Tips and Tricks for containerizing services
===========================================

This document contains a list of tips and tricks that are useful when
containerizing an OpenStack service.

Monitoring containers
---------------------

It's often useful to monitor the running containers and see what has been
executed and what not. The puppet containers are created and removed
automatically unless they fail. For all the other containers, it's enough to
monitor the output of the command below::

    $ watch -n 0.5 docker ps -a --filter label=managed_by=docker-cmd

.. _debug-containers:

Viewing container logs
----------------------

You can view the output of the main process running in a container by running::

    $ docker logs $CONTAINER_ID_OR_NAME

Ideally all containerized processes would log everything to
stdout/stderr and the above command would suffice. Not all services
are quite there yet, so we export traditional logs from containers
into the `/var/log/containers` directory on the host, where you can
look at them.

Debugging container failures
----------------------------

The following commands are useful for debugging containers.

* **inspect**: This command allows for inspecting the container's structure and
  metadata. It provides info about the bind mounts on the container, the
  container's labels, the container's command, etc::

    $ docker inspect $CONTAINER_ID_OR_NAME

  There's no shortcut for *rebuilding* the command that was used to run the
  container but, it's possible to do so by using the `docker inspect` command
  and the format parameter::

   $ docker inspect --format='{{range .Config.Env}} -e "{{.}}" {{end}} {{range .Mounts}} -v {{.Source}}:{{.Destination}}{{if .Mode}}:{{.Mode}}{{end}}{{end}} -ti {{.Config.Image}}' $CONTAINER_ID_OR_NAME

  Copy the output from the command above and append it to the one below, which
  will run the same container with a random name and remove it as soon as the
  execution exits::

    $ docker run --rm $OUTPUT_FROM_PREVIOUS_COMMAND /bin/bash

* **exec**: Running commands on or attaching to a running container is extremely
  useful to get a better understanding of what's happening in the container.
  It's possible to do so by running the following command::

    $ docker exec -ti $CONTAINER_ID_OR_NAME /bin/bash

  Replace the `/bin/bash` above with other commands to run oneshot commands. For
  example::

    $ docker exec -ti mysql mysql -u root -p $PASSWORD

  The above will start a mysql shell on the mysql container.

* **export** When the container fails, it's basically impossible to know what
  happened. It's possible to get the logs from docker but those will contain
  things that were printed on the stdout by the entrypoint. Exporting the
  filesystem structure from the container will allow for checking other logs
  files that may not be in the mounted volumes::

    $ docker export $CONTAINER_ID_OR_NAME | tar -C /tmp/$CONTAINER_ID_OR_NAME -xvf -


Debugging with Paunch
---------------------

The ``paunch debug`` command allows you to perform specific actions on a given
container.  This can be used to:

* Run a container with a specific configuration.
* Dump the configuration of a given container in either json or yaml.
* Output the docker command line used to start the container.
* Run a container with any configuration additions you wish such that you can
  run it with a shell as any user etc.

The configuration options you will likely be interested in here include:

::

  --file <file>         YAML or JSON file containing configuration data
  --action <name>       Action can be one of: "dump-json", "dump-yaml",
                        "print-cmd", or "run"
  --container <name>    Name of the container you wish to manipulate
  --interactive         Run container in interactive mode - modifies config
                        and execution of container
  --shell               Similar to interactive but drops you into a shell
  --user <name>         Start container as the specified user
  --overrides <name>    JSON configuration information used to override
                        default config values

``file`` is the name of the configuration file to use
containing the configuration for the container you wish to use.
TripleO creates configuration files for starting containers in
``/var/lib/tripleo-config/``.  If you look in this directory
you will see a number of files corresponding with the steps in
TripleO heat templates.  Most of the time, you will likely want to use
``/var/lib/tripleo-config/hashed-docker-container-startup-config-step_4.json``
as it contains most of the final startup configurations for the running
containers.

To make sure you get the right container you can use the ``paunch list``
command to see what containers are running and which config id they
are using.  This config id corresponds to which file you will find the
container configuration in.

Here is an example of using ``paunch debug`` to start a root shell inside the
heat api container:

::

  # paunch debug --file /var/lib/tripleo-config/hashed-docker-container-startup-config-step_4.json --interactive --shell --user root --container heat_api --action run

This will drop you an interactive session inside the heat api container
starting /bin/bash running as root.

To see how this container is started by TripleO:

::

  # paunch debug --file /var/lib/tripleo-config/hashed-docker-container-startup-config-step_4.json --container heat_api --action print-cmd

  docker run --name heat_api-t7a00bfz --detach=true --env=KOLLA_CONFIG_STRATEGY=COPY_ALWAYS --env=TRIPLEO_CONFIG_HASH=b3154865d1f722ace643ffbab206bf91 --net=host --privileged=false --restart=always --user=root --volume=/etc/hosts:/etc/hosts:ro --volume=/etc/localtime:/etc/localtime:ro --volume=/etc/puppet:/etc/puppet:ro --volume=/etc/pki/ca-trust/extracted:/etc/pki/ca-trust/extracted:ro --volume=/etc/pki/tls/certs/ca-bundle.crt:/etc/pki/tls/certs/ca-bundle.crt:ro --volume=/etc/pki/tls/certs/ca-bundle.trust.crt:/etc/pki/tls/certs/ca-bundle.trust.crt:ro --volume=/etc/pki/tls/cert.pem:/etc/pki/tls/cert.pem:ro --volume=/dev/log:/dev/log --volume=/etc/ssh/ssh_known_hosts:/etc/ssh/ssh_known_hosts:ro --volume=/var/lib/kolla/config_files/heat_api.json:/var/lib/kolla/config_files/config.json:ro --volume=/var/lib/config-data/heat_api/etc/heat/:/etc/heat/:ro --volume=/var/lib/config-data/heat_api/etc/httpd/conf/:/etc/httpd/conf/:ro --volume=/var/lib/config-data/heat_api/etc/httpd/conf.d/:/etc/httpd/conf.d/:ro --volume=/var/lib/config-data/heat_api/etc/httpd/conf.modules.d/:/etc/httpd/conf.modules.d/:ro --volume=/var/lib/config-data/heat_api/var/www/:/var/www/:ro --volume=/var/log/containers/heat:/var/log/heat 192.168.24.1:8787/tripleoupstream/centos-binary-heat-api:latest

You can also dump the configuration of this to a file so you can edit
it and rerun it with different a different configuration:

::

  # paunch debug --file /var/lib/tripleo-config/hashed-docker-container-startup-config-step_4.json --container heat_api --action dump-json > heat_api.json

You can then use ``heat_api.json`` as your ``--file`` argument after
editing it to your liking.

You can also add any configuration elements you wish on the command line
to test paunch or debug containers etc.  In this example I'm adding a
health check to the container:

::

  # paunch debug --file /var/lib/tripleo-config/hashed-docker-container-startup-config-step_4.json --overrides '{"health-cmd": "/usr/bin/curl -f http://localhost:8004/v1/", "health-interval": "30s"}' --container heat_api --action run
  172ed68eb44ab20551a70a3e33c90a02014f530e42cd7b30255da4577c8ed80c


Debugging docker-puppet.py
--------------------------

The :ref:`docker-puppet.py` script manages the config file generation and
puppet tasks for each service.  This also exists in the `docker` directory
of tripleo-heat-templates.  When writing these tasks, it's useful to be
able to run them manually instead of running them as part of the entire
stack. To do so, one can run the script as shown below::

  CONFIG=/path/to/task.json /path/to/docker-puppet.py

The json file must follow the following form::

    [
        {
            "config_image": ...,
            "config_volume": ...,
            "puppet_tags": ...,
            "step_config": ...
        }
    ]


Using a more realistic example. Given a `puppet_config` section like this::

      puppet_config:
        config_volume: glance_api
        puppet_tags: glance_api_config,glance_api_paste_ini,glance_swift_config,glance_cache_config
        step_config: {get_attr: [GlanceApiPuppetBase, role_data, step_config]}
        config_image:
          list_join:
            - '/'
            - [ {get_param: DockerNamespace}, {get_param: DockerGlanceApiImage} ]


Would generated a json file called `/var/lib/docker-puppet-tasks2.json` that looks like::

    [
        {
            "config_image": "tripleoupstream/centos-binary-glance-api:latest",
            "config_volume": "glance_api",
            "puppet_tags": "glance_api_config,glance_api_paste_ini,glance_swift_config,glance_cache_config",
            "step_config": "include ::tripleo::profile::base::glance::api\n"
        }
    ]


Setting the path to the above json file as value to the `CONFIG` var passed to
`docker-puppet.py` will create a container using the
`centos-binary-glance-api:latest` image and it and run puppet on a catalog
restricted to the given puppet `puppet_tags`.

As mentioned above, it's possible to create custom json files and call
`docker-puppet.py` manually, which makes developing and debugging puppet steps
easier.

`docker-puppet.py` also supports the environment variable `SHOW_DIFF`,
which causes it to print out a docker diff of the container before and
after the configuration step has occurred.

By default `docker-puppet.py` runs things in parallel.  This can make
it hard to see the debug output of a given container so there is a
`PROCESS_COUNT` variable that lets you override this.  A typical debug
run for docker-puppet might look like::

    SHOW_DIFF=True PROCESS_COUNT=1 ./docker-puppet.py

Testing in CI
-------------

When new service containers are added, ensure to update the image names in
`container-images/overcloud_containers.yaml` tripleo-common repo. These service
images are pulled in and available in the local docker registry that the
containers ci job uses::

    uploads:
        - imagename: tripleoupstream/centos-binary-example:latest
