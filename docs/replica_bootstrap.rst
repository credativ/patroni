.. _replica_imaging_and_bootstrap:

Replica imaging and bootstrap
=============================

Patroni allows customizing creation of a new replica. It also supports defining what happens when the new empty cluster
is being bootstrapped. The distinction between two is well defined: Patroni creates replicas only if the ``initialize``
key is present in DCS for the cluster. If there is no ``initialize`` key - Patroni calls bootstrap exclusively on the
first node that takes the initialize key lock.

.. _custom_bootstrap:

Bootstrap
---------

PostgreSQL provides ``initdb`` command to initialize a new cluster and Patroni calls it by default. In certain cases,
particularly when creating a new cluster as a copy of an existing one, it is necessary to replace a built-in method with
custom actions. Patroni supports executing user-defined scripts to bootstrap new clusters, supplying some required
arguments to them, i.e. the name of the cluster and the path to the data directory. This is configured in the
``bootstrap`` section of the Patroni configuration. For example:

.. code:: YAML

    bootstrap:
        method: <custom_bootstrap_method_name>
        <custom_bootstrap_method_name>:
            command: <path_to_custom_bootstrap_script> [param1 [, ...]]
            keep_existing_recovery_conf: False
            no_params: False
            recovery_conf:
                recovery_target_action: promote
                recovery_target_timeline: latest
                restore_command: <method_specific_restore_command>


Each bootstrap method must define at least a ``name`` and a ``command``. A special ``initdb`` method is available to trigger
the default behavior, in which case ``method`` parameter can be omitted altogether. The ``command`` can be specified using either
an absolute path, or the one relative to the ``patroni`` command location. In addition to the fixed parameters defined
in the configuration files, Patroni supplies two cluster-specific ones:

--scope
    Name of the cluster to be bootstrapped
--datadir
    Path to the data directory of the cluster instance to be bootstrapped

Passing these two additional flags can be disabled by setting a special ``no_params`` parameter to ``True``.

If the bootstrap script returns ``0``, Patroni tries to configure and start the PostgreSQL instance produced by it. If any
of the intermediate steps fail, or the script returns a non-zero value, Patroni assumes that the bootstrap has failed,
cleans up after itself and releases the initialize lock to give another node the opportunity to bootstrap.

If a ``recovery_conf`` block is defined in the same section as the custom bootstrap method, Patroni will generate a
``recovery.conf`` before starting the newly bootstrapped instance (or set the recovery settings on Postgres configuration if
running PostgreSQL >= 12).
Typically, such recovery configuration should contain at least one of the ``recovery_target_*`` parameters, together with the ``recovery_target_action`` set to ``promote``.

If ``keep_existing_recovery_conf`` is defined and set to ``True``, Patroni will not remove the existing ``recovery.conf`` file if it exists (PostgreSQL <= 11).
Similarly, in that case Patroni will not remove the existing ``recovery.signal`` or ``standby.signal`` if either exists, nor will it override the configured recovery settings (PostgreSQL >= 12).
This is useful when bootstrapping from a backup with tools like pgBackRest that generate the appropriate recovery configuration for you.

Besides that, any additional key/value pairs informed in the custom bootstrap method configuration will be passed as arguments to ``command`` in the format ``--name=value``. For example:

.. code:: YAML

    bootstrap:
        method: <custom_bootstrap_method_name>
        <custom_bootstrap_method_name>:
            command: <path_to_custom_bootstrap_script>
            arg1: value1
            arg2: value2

Makes the configured ``command`` to be called additionally with ``--arg1=value1 --arg2=value2`` command-line arguments.

 .. note:: Bootstrap methods are neither chained, nor fallen-back to the default one in case the primary one fails

As an example, you are able to bootstrap a fresh Patroni cluster from a Barman backup with a configuration like this:

.. code:: YAML

    bootstrap:
        method: barman
        barman:
            keep_existing_recovery_conf: true
            command: patroni_barman --api-url https://barman-host:7480 recover
            barman-server: my_server
            ssh-command: ssh postgres@patroni-host

.. note::
    ``patroni_barman recover`` requires that you have both Barman and ``pg-backup-api`` configured in the Barman host, so it can execute a remote ``barman recover`` through the backup API.
    The above example uses a subset of the available parameters. You can get more information running ``patroni_barman recover --help``.

.. _custom_replica_creation:

Building replicas
-----------------

Patroni uses tried and proven ``pg_basebackup`` in order to create new replicas. One downside of it is that it requires
a running leader node. Another one is the lack of 'on-the-fly' compression for the backup data and no built-in cleanup
for outdated backup files. Some people prefer other backup solutions, such as ``WAL-E``, ``pgBackRest``, ``Barman`` and
others, or simply roll their own scripts. In order to accommodate all those use-cases Patroni supports running custom
scripts to clone a new replica. Those are configured in the ``postgresql`` configuration block:

.. code:: YAML

    postgresql:
        create_replica_methods:
            - <method name>
        <method name>:
            command: <command name>
            keep_data: True
            no_params: True
            no_leader: 1

example: wal_e

.. code:: YAML

    postgresql:
        create_replica_methods:
            - wal_e
            - basebackup
        wal_e:
            command: patroni_wale_restore
            no_leader: 1
            envdir: {{WALE_ENV_DIR}}
            use_iam: 1
        basebackup:
            max-rate: '100M'

example: pgbackrest

.. code:: YAML

    postgresql:
        create_replica_methods:
            - pgbackrest
            - basebackup
        pgbackrest:
            command: /usr/bin/pgbackrest --stanza=<scope> --delta restore
            keep_data: True
            no_params: True
        basebackup:
            max-rate: '100M'

example: Barman

.. code:: YAML

    postgresql:
        create_replica_methods:
            - barman
            - basebackup
        barman:
            command: patroni_barman --api-url https://barman-host:7480 recover
            barman-server: my_server
            ssh-command: ssh postgres@patroni-host
        basebackup:
            max-rate: '100M'

.. note::
    ``patroni_barman recover`` requires that you have both Barman and ``pg-backup-api`` configured in the Barman host, so it can execute a remote ``barman recover`` through the backup API.
    The above example uses a subset of the available parameters. You can get more information running ``patroni_barman recover --help``.

The ``create_replica_methods`` defines available replica creation methods and the order of executing them. Patroni will
stop on the first one that returns 0. Each method should define a separate section in the configuration file, listing the command
to execute and any custom parameters that should be passed to that command. All parameters will be passed in a
``--name=value`` format. Besides user-defined parameters, Patroni supplies a couple of cluster-specific ones:

--scope
    Which cluster this replica belongs to
--datadir
    Path to the data directory of the replica
--role
    Always 'replica'
--connstring
    Connection string to connect to the cluster member to clone from (primary or other replica). The user in the
    connection string can execute SQL and replication protocol commands.

A special ``no_leader`` parameter, if defined, allows Patroni to call the replica creation method even if there is no
running leader or replicas. In that case, an empty string will be passed in a connection string. This is useful for
restoring the formerly running cluster from the binary backup.

A special ``keep_data`` parameter, if defined, will instruct Patroni to  not clean PGDATA folder before calling restore.

A special ``no_params`` parameter, if defined, restricts passing parameters to custom command.

A ``basebackup`` method is a special case: it will be used if
``create_replica_methods`` is empty, although it is possible
to list it explicitly among the ``create_replica_methods`` methods. This method initializes a new replica with the
``pg_basebackup``, the base backup is taken from the leader unless there are replicas with ``clonefrom`` tag, in which case one
of such replicas will be used as the origin for pg_basebackup. It works without any configuration; however, it is
possible to specify a ``basebackup`` configuration section. Same rules as with the other method configuration apply,
namely, only long (with --) options should be specified there. Not all parameters make sense, if you override a connection
string or provide an option to created tar-ed or compressed base backups, patroni won't be able to make a replica out
of it. There is no validation performed on the names or values of the parameters passed to the ``basebackup`` section.
Also note that in case symlinks are used for the WAL folder it is up to the user to specify the correct ``--waldir``
path as an option, so that after replica buildup or re-initialization the symlink would persist. This option is supported
only since v10 though.

You can specify basebackup parameters as either a map (key-value pairs) or a list of elements, where each element
could be either a key-value pair or a single key (for options that does not receive any values, for instance, ``--verbose``).
Consider those 2 examples:

.. code:: YAML

    postgresql:
        basebackup:
            max-rate: '100M'
            checkpoint: 'fast'

and

.. code:: YAML

    postgresql:
        basebackup:
            - verbose
            - max-rate: '100M'
            - waldir: /pg-wal-mount/external-waldir

If all replica creation methods fail, Patroni will try again all methods in order during the next event loop cycle.
