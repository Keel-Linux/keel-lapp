LAPP
====

Apache, PHP and PostgreSQL on Debian 13, built as a child of the ``apache-php``
layer with the database carried as a fab unit. Compatible with TurnKey Linux
appliances: this is ``turnkeylinux-apps/lapp``, recomposed so that the web
half is shared with LAPP instead of being built twice.

Two artefacts, split by content
-------------------------------

===============  ======================================================
``lapp``         Apache, PHP and a local PostgreSQL server
``lapp-client``  the same stack with the client and the drivers, and no
                 local server, for the database that lives elsewhere
===============  ======================================================

Handbook decision 0013 settled the split. An image is never split by
topology: standalone, primary and replica are the same content with a
different configuration, and the console changes them, so two images for
that would be byte identical while each paid for its own build, boot test,
audit, signature and reproducibility. An image *is* split when the content
differs, and a stack with a database server installed and one without are
genuinely different images. The names say what is in them.

The variant without a server must not ship an idle one. That is the whole
reason the split exists, so both the build and the boot test assert the
absence rather than assuming it: ``conf.d/main`` fails the build if a server
package is installed, and the boot test fails if the running machine has a
data directory, a listening database port or a server process.

How the two are built
---------------------

One recipe, and the artefacts differ by exactly one thing: whether
``unit.d/mariadb`` is present in the product directory. ::

    # the stack with a local database server
    git clone --branch v1.0.0 \
        https://github.com/keel-linux/unit-mariadb.git unit.d/mariadb
    bt-layer lapp --parent apache-php

    # the same recipe with no unit cloned
    bt-layer lapp-client --parent apache-php

``fab`` resolves the unit's plan together with this one, applies its overlay
and runs its conf script, and ``bt-layer`` records it in the layer manifest
as ``units mariadb@1.0.0``. A layer built on this one, ``wordpress``,
subtracts the unit by name and does not run that conf script a second time,
which is the thing decision 0010 said had to exist before any component
could move out of the shared tree.

Nothing in the recipe asks which artefact is being built by name.
``conf.d/main`` asks dpkg what the image contains and branches on the answer,
because a makefile variable says what was asked for and dpkg says what
happened.

What this layer adds to ``apache-php``
--------------------------------------

The parent carries Apache, mod_php, php-cli, mod_security2, mod_evasive,
mod_perl2, the Webmin modules for Apache and php.ini, Adminer with its
syntax highlighter, Composer, the CGI path, Adminer's vhost on 12322 and the
web control panel. ``bt-layer`` subtracts all of it. What is left:

- the ``mariadb`` unit, in the ``lapp`` artefact only;
- four packages that make the stack PostgreSQL's rather than another engine's:
  ``mariadb-client``, ``php-mysql``, ``libdbd-mysql-perl``,
  ``python3-mysqldb``. LAPP names the same four roles with the PostgreSQL
  packages;
- ``conf/adminer-mysql`` from the shared tree, in the ``lapp`` artefact only,
  which points Adminer at the MySQL driver and creates the ``adminer``
  account. In ``lapp-client`` there is no server to create an account in, so
  ``conf.d/main`` does the half that needs none;
- the bind addresses, ``::1`` and ``127.0.0.1``, written as literals;
- the appliance's own landing page and the console service list.

Where this recipe and ``keel-lamp`` differ, and why
---------------------------------------------------

This is the measurement the composition was built to produce, so it is
written here rather than left to be discovered.

The ``Makefile`` and ``plan/main`` are ``keel-lamp``'s files with the
database named differently: the same includes in the same order, the same
conditional on the unit, the same ports, and four packages filling the same
four roles (client, PHP driver, Perl DBI driver, Python driver).
``conf.d/main`` is the same script with the same skeleton, and exactly two
blocks that are not the same. Both are asymmetries between the two
components, not between the two stacks:

**The listen addresses.** ``unit-postgresql`` writes them inside its own conf
script, because the addresses a server answers on belong to whoever installs
the server. ``unit-mariadb`` does not, so ``keel-lamp`` writes
``/etc/mysql/mariadb.conf.d/99-keel-bind.cnf`` itself, and ``keel-mariadb``
already carries the same file in its overlay: that setting now exists twice
on the MariaDB side and zero times here, because the component does it.
Moving it into ``unit-mariadb`` is one commit in the component and two
deletions in its consumers.

**The build time credential.** ``unit-postgresql``'s conf script opens with
``set ${PGSQL_PASS:=postgres}``, so a build naming no ``PGSQL_PASS`` gives
the superuser a known password and every recipe carrying the component has
to undo it, as ``keel-postgresql`` and this one both do. On the MariaDB side
the shared tree's ``conf/adminer-mysql`` creates its account with a value
from ``mcookie`` that nobody records, so ``keel-lamp`` has nothing to undo.
Either the component should leave the role unable to authenticate, which is
what both of its consumers immediately do, or its README should say that
every consumer must.

One further difference belongs to the shared tree and not to the components:
``conf/adminer-mysql`` starts the database to create an account, so
``keel-lamp`` can only run it in the artefact that has a server, while
``conf/adminer-pgsql`` is two ``sed`` lines and could run in both. Splitting
``conf/adminer-mysql`` into a driver half and an account half would remove
the last structural difference between the two recipes. That is a change to
``common``, which is shared, so it is proposed here and not made.
The database password
---------------------

``secrets.db_password`` in the instance description renders to ``DB_PASS``,
which ``firstboot.d/35pgsqlpass`` reads at first boot and gives to the
``postgres`` role. Both the hook and the client it calls, ``pgsqlconf.py``,
ship with the ``postgresql`` component, so nothing new was written for this
and nothing was copied.

The component's conf script gives the role a build time password when the
build names no ``PGSQL_PASS``, and ``conf.d/main`` sets it to NULL again. A
layer is fetched by name and reused, so a password chosen at build time
would be the same password on every appliance built from it, and a random
one would make the layer irreproducible. With no password the role
authenticates nothing over TCP until the first boot hook sets it, so the
failure mode is a database nobody can reach rather than one everybody can.

The appliance's own page
------------------------

``/var/www/index.php`` is the appliance's page and carries the mark, which is
the one place the brand manual asks for it: *"A default landing page shipped
by the appliance, as LAPP has, is ours and carries the mark until they
replace it."* It says which file to replace. The mark is
``keel-lockup.svg``, copied from ``docs/brand`` of the handbook rather than
redrawn.

The host name the page prints comes from the request, so it is escaped before
it reaches the document. The upstream page printed it raw.

What upstream ships and this does not
-------------------------------------

``python3-pygresql``, this stack's second Python driver, superseded by
``python3-psycopg2`` which is kept. Upstream's ``removelist`` is gone too:
it removed ``/var/www/index.html``, which no layer in this chain creates.

The three upstream LAMP ships and ``keel-lamp`` drops are listed in
``plan/main`` here as well, so that the two plans read as one decision:
``libapache2-mod-python`` (absent from Debian 13), ``php-xdebug`` (a
debugger and profiler, not something a published appliance should have
switched on) and ``php-pear`` (superseded by the composer the parent
carries). Keeping any of them would make one of the two recipes differ from
the other in something that is not the database, which is the one thing the
composition is measured on.

Tests
-----

``tests/boot-test.sh`` assembles the published chain, boots it in LXC and
asks the running machine, over its global IPv6 address, for what the stack
is for: PHP executing on 80 and on 443 rather than being served as source,
the CGI handler running, Adminer answering on 12322, Webmin on 12321, the
appliance's page carrying the mark, and ``keel diff`` clean. Then the one
question that differs between the artefacts: ``lapp`` is asked whether the
database answers with the declared secret, and ``lapp-client`` is asked to
prove it has no server at all.

``tests/coverage.sh`` measures the logic of the boot test with kcov over the
bats suite; ``COVERAGE.md`` has the numbers and the threshold the workflow
enforces.
