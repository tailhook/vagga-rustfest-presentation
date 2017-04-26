:css: my.css

.. role:: kill
   :class: kill

Vagga
=====

----

The Plan
========

* What is vagga?
* Challenges
* Rust + Containers
* Deployment Tools

----

What is Vagga?
==============


----

vagga is...

========================
venv/nvm/rvm on Steroids
========================

----

.. code-block:: yaml

    containers:
      assets:
        setup:
        - !Ubuntu xenial
        - !Install [imagemagick, make]
        - !NpmDependencies "package.json"

----

.. code-block:: yaml

    commands:
      webpack: !Command
        container: assets
        run: [webpack]
      run: !Command
        container: assets
        environ: { FLASK_APP: "myapp.py" }
        run: "python -m flask run"

----

.. code-block:: console

    $ git clone git://github.com/.../foobar
    $ cd foobar
    $ vagga
    Available commands:
        build       Build static files
        run         Run nginx+app+redis
        build-docs  Build docs
    $ vagga build

----

.. code-block:: console

    $ git pull
    $ vagga run

----

vagga is...

================================
The Higher Level Package Manager
================================

----

.. code-block:: yaml

  nginx:
    setup:
    - !Alpine v3.5
    - !Install [nginx]
    - !Build
      container: jsstatic
      source: /var/javascripts
      path: /srv/www

----

.. code-block:: yaml

  run: !Command
    container: rust
    prerequisites: [make-bin, make-js]
    run: "./target/debug/app"

----

:id: supervise

.. code-block:: yaml

  run: !Supervise
    description: Run full server stack
    children:
      redis: !Command
        container: redis
        run: [redis-server, --daemonize, no]
      nginx: !Command
        container: nginx
        run: [nginx, -c, /work/config/nginx.conf]
      foobar: !Command
        run: [python, -m, foobar]


----

vagga is...

=======================================
A Containerization Tool Without Daemons
=======================================

----

::

     \-+= vagga run
       |-+= python -m foobar
       |-+= redis-server --daemonize --no
       \-+= nginx -c /work/config/nginx.conf

----

:id: dockertree

::

   # docker tree
   -+= 00001 root systemd --system
    |-+- 10771 root docker -d
    | \--= 32029 root bash   << our process
    \-+= 30029 pc tmux
      \-+= 10718 pc -zsh     << our shell
        \--= 32021 pc docker run -it --rm bash

::

   # vagga tree
   -+= 00001 root systemd --system
    \-+= 30029 pc tmux
      \-+= 10358 pc -zsh        << our shell
        \-+= 00940 pc vagga bash
          \-+- 00941 pc vagga bash
            \--= 00942 pc bash  << our process

----

Vagga
=====

* simple YAML config (+versioning)
* user namespaces (no root/setuid)
* multiple process monitoring
* only for dev.env.

(written in rust)

----

Challenges
==========

----

PID1
====

* KILL
* Signals
* Reparenting

(remember ``tini``?)

----

After Clone
===========

* No memory allocations
* more things allocate in debugging version than in release

-----

Cloexec
=======

* Cloexec by default

-----

Rust
====

-----

Unshare Crate
=============

-----

.. code-block:: rust

   Command::new("sh")
   .arg("-c").arg("echo hello")
   .status().unwrap()

-----

.. code-block:: rust

   Command::new("sh")
   .arg("-c").arg("echo hello")
   .unshare(&[Net, User, Uts, Mount])
   .chroot_dir("/container")
   .set_id_maps(...)
   .status().unwrap()

-----

Libmount Crate
==============

-----

.. code-block:: rust

   Tmpfs::new("/tmp")
   .size_bytes(1_048_576)
   .mount()


-----

.. code-block:: rust

    let src = "/x";
    let dest = "/y";
    Bind::new(&src, &dest)
    .bare_mount()
    .map_err(|e| format!(
        "bind mount {:?} -> {:?}: {}",
        src, dest, e))?

-----

:id: error1

::

    Fatal error: Can't mount bind /x to /y:
        No such file or directory (os error 2)

-----

.. code-block:: rust

   Bind::new("/x", "/y")
       .mount()?

-----

:id: error2

::

    Fatal error: recursive bind mount "/x" -> "/y":
        No such file or directory (os error 2)
        (source: exists, target: missing, superuser)

-----

To Do
=====

----

Get rid of busybox:

* :kill:`tar/unzip`
* Use Tokio to download files
* ``ip``
* ``iptables``
* ``brctl``


-----

Deployment
==========

-----

* lithos -- containers
* cantal -- monitoring
* verwalter -- orchestration

All three written in rust

-----

Lithos
======

* Uses unshare + libmount crates
* Super simple, < 5k LoC


-----

Cantal
======

*

* cantal -- monitoring
* verwalter -- orchestration
