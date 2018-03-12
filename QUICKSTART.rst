Quick Start Guide
=================


Bootstrap
---------

Let's say you have a python service ``hello.py``:

.. code-block:: python

    import asyncio
    from aiohttp import web

    async def hello(request):
        return web.Response(body=b"Hello, world", content_type="text/html")

    app = web.Application()
    app.router.add_route('GET', '/', hello)
    web.run_app(app)

And a ``vagga.yaml`` for it:

.. code-block:: yaml

    containers:

      py-base:
        setup:
        - !Alpine v3.7
        - !PipConfig { dependencies: true }
        - !Py3Install [aiohttp]

    commands:

      run: !Command
        container: py-base
        description: Run the application
        run: [python3, hello.py]

Check it with:

.. code-block:: console

    > vagga run
    WARN: 2018-03-06T00:19:43Z: vagga::builder::guard: Indexing container...
    WARN: 2018-03-06T00:19:43Z: vagga::wrapper::build: Container py-base (b8306540) built in 23 seconds.
    ======== Running on http://0.0.0.0:8080 ========
    (Press CTRL+C to quit)

And remember to add ignore list:

.. code-block:: console

    > echo /.vagga >> .gitignore
    > git add .
    > git commit -a -m "initial commit"


Setting Up Machine
==================

Any Ubuntu Xenial-based machine will work as a base system, and you need root
access to the system. Spawning a VM on ssh'shing to it on AWS, GCE or
Digitalocean should work well.

In root SSH shell add mintaka's package list and install mintaka package:

.. code-block:: console

    > curl https://raw.githubusercontent.com/mglawica/mintaka/master/mintaka-ubuntu-xenial.list | sudo tee /etc/apt/sources.list.d/mintaka.list
    > apt-get update && apt-get install -y mintaka
    [ .. snip .. ]
    Setting up cantal (0.5.11+xenial1) ...
    Setting up ciruela (0.5.11+xenial1) ...
    Setting up lithos (0.15.1+xenial1) ...
    Setting up verwalter (0.11.0+xenial1) ...
    Setting up swindon (0.7.8+xenial1) ...
    Setting up mintaka (0.1.0+xenial1) ...

First time after installation you should start the services:

.. code-block:: console

    > systemctl daemon-reload
    > systemctl start multi-user.target

All of the infrastructure services listen on local by default, so
to connect to it from your own machine use ssh tunelling. Log out from
the current shell and call ssh again like this:

.. code-block:: console

    > ssh \
        -L 8379:127.0.0.1:8379 \
        -L 22682:127.0.0.1:22682 \
        -L 24783:127.0.0.1:24783 \
        root@<your host ip>
    root@your-vm:~#

This will bring you to a normal ssh shell, but as long as it is active you
can connect to services on remote system by accessing those ports locally.
You can put ``LocalForward`` directives to ``~/.ssh/config`` to avoid copying
this long command-line each time.

.. hint:: You don't need to keep ssh connected to keep services running, you
   just need it to connect to web UI (and to deploy if you're deploying from
   your own system) and you can safely reconnect at any time.

(alternatively change ``PRIVATE_IP`` in ``/etc/mintaka.env`` and setup a
VPN or firewall, but that is out of the scope of this tutorial).


First Deploy
============

You need to create a key for deployment:

.. code-block:: console

    > ssh-keygen -f .vagga/key -t ed25519 -N ''
    Generating public/private ed25519 key pair.
    Your identification has been saved in .vagga/key.
    Your public key has been saved in .vagga/key.pub.

Remember to keep ``.vagga/key`` in secret (and don't commit to git).

Now copy the public file to the clipboard, for example like this:

.. code-block:: console

    > xsel -b < .vagga/key.pub

And create a new "source" at ``http://localhost:8379/settings/sources/new``,
let's name it ``hello1``.

.. hint:: If you're deploying from CI put private key into
   ``VAGGAENV_CIRUELA_KEY`` and remove from local disk.

Now you can prepare your repository for the first deploy.

First add a container that is inherited from your base python container and
contains the actual application code (which we run from current directory
in a development mode). Here is a piece of ``vagga.yaml``:

.. code-block:: yaml

    containers:
      py-base:   # ... already there ...

      py:
        setup:
        - !SubConfig
          path: vagga.yaml
          container: py-base  # original container
        - !EndureDir /app
        - !Copy
          source: /work/hello.py
          path: /app/hello.py

We also need deployment command:

.. code-block:: yaml

    commands:
      run:   # ... already there ...

      deploy: !CapsuleCommand
        description: Run wark deployment tool
        run:
        - vagga
        - _capsule
        - script
        - https://github.com/mglawica/wark/releases/download/v0.3.3/wark
        - --destination=http://localhost:8379/~kk/files/generic.yaml
        - -Dproject=hello1    # the name of the source you've just added
        - -Dhost=localhost


Our deployment tool "wark" also adds some stuff to the containers, so we need
to refer to it's own vagga file. Just add somewhere near the top of
``vagga.yaml``:

.. code-block:: yaml

    mixins:
    - vagga/deploy.yaml


(first time you run vagga it will warn you that no such mixin exists, that's
fine we'll create it shortly).

Now if you've done everything right you should see the following:

.. code-block:: console

    > vagga deploy update
    [ .. other warnings snipped .. ]
    ERROR: No deployments available

This means we need to add configs for actual services. Let's put the following
into a file ``config/deploy-prod/lithos.web.yaml``:

.. code-block:: yaml

    kind: Daemon
    metadata:
      container: py       # name of container in vagga.yaml
    memory-limit: 100Mi
    workdir: /app
    executable: /usr/bin/python3
    arguments:
    - "hello.py"

.. note:: the convention ``config/deploy-<KIND>/lithos.<NAME>.yaml`` is
   encoded in the wark configuration file. If you're deploying to other
   destination convention might be different.

Now we can generate configs:

.. code-block:: console

    > vagga deploy update
    [ .. snip .. ]
     INFO 2018-03-12T23:39:06Z: wark::local: All done, ready for deploy

It's a good idea to commit everthing done so far and tag it:

.. code-block:: console

    > git add .
    > git commit -m "Initial deployment configs"
    > git tag -a v0.1.0 -m "Initial release"

(you might also want to push branch and tag now)

Okay finally we can push files to the server. You need to pass key file by
environment variable:

.. code-block:: console

    > VAGGAENV_CIRUELA_KEY=$(cat .vagga/key) vagga deploy -d prod
    [ .. snipped container building and uploading .. ]
     INFO 2018-03-09T00:24:26Z: wark::deploy: Version "v0.1.0" is successfully deployed


Configuring Services
====================

You need some GUI work to start the server now. Go to http://localhost:8379/settings/projects :

1. Add new project
2. Add a group to the project
3. Add your service to the group, your service config name and version should
   already be in the UI

That's it. Now you can visit your ``http://<server-ip>:8080/`` to see the
service.


Updating Code
=============

Just run the:

.. code-block:: console

    > VAGGAENV_CIRUELA_KEY=$(cat .vagga/key) vagga deploy -d prod

This is the same line you need to add to your CI system (and probably
``VAGGAENV_CIRUELA_KEY`` hidden somewhere in CI variables UI).


