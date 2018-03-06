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

    > systemctl daemon-reload
    > systemctl start multi-user.target

All of the infrastructure services listen on local by default, so
to connect to it from your own machine use ssh tunelling. Log out from
the current shell and call ssh again like this:

    > ssh \
        -L localhost:8379:127.0.0.1:8379 \
        -L localhost:22682:127.0.0.1:22682 \
        -L localhost:24783:127.0.0.1:24783 \
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

And create a new "source" at ``http://localhost:8379/~kk/sources/new``,
let's name it ``hello1``.

.. hint:: If you're deploying from CI put private key into
   ``VAGGAENV_CIRUELA_KEY`` and remove from local disk.
