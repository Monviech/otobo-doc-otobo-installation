Migration from OTRS 6 or OTRS 7 / ((OTRS)) Community Edition to OTOBO version 10.1
==================================================================================

.. warning::
    Please migrate your OTRS to OTOBO version 10.1 first and then upgrade your OTOBO to version 11. 

Welcome and thank you for choosing OTOBO!

OTRS, ((OTRS)) Community Edition and OTOBO are very comprehensive and flexible in their application. Thus, every migration to OTOBO requires thorough preparation and possibly some rework, too.

Please take your time for the migration and follow these instructions step by step.

If you have any problem or question, please do not despair. Call our support line, write an email, or post your query
in the OTOBO Community forum at https://forum.otobo.org/. We will find a way to help you!

.. note::

    After the migration the data previously available in OTRS will be available in OTOBO 10.
    We do not modify any data of the OTRS installation during the migration.

Overview over the Supported Migration Szenarios
------------------------------------------------

With the OTOBO Migration Interface it is possible to employ the following migration strategies:

1.  The general migration strategy.

    This is the regular way to perform a migration. Many different different combinations are supported:

    Change server:
        Migrate and simultaneously move to a new application server.

    Separate application and web servers:
        It's your choice whether you want to run application and database server on
        the same host or each on a dedictated host. This choice is regardless of the previous setup in OTRS / ((OTRS)) Community Edition.

    Different databases:
        Migrate from any of the supported databases to any other supported database.

    Different operating system:
        Switch from any supported operating system to any other supported operating system.

    Docker:
        Migrate to a Docker-based installation of OTOBO 10.

2.  A variant of the general strategy where the database migration is streamlined.

    Use the ETL-like migration when the source database mustn't suffer from increased load
    or when access to the source database is a bottleneck. In the general strategy, the data is row by row
    first read from the *otrs* database and then inserted into the OTOBO database.
    In this variant, the complete *otrs* database tables are first exported, then transformed,
    and then imported into the *otobo* database.

3.  Migration from an Oracle based OTRS 6 / OTRS 7 installation to an Oracle based OTOBO installation.

    This is a special case that is not supported by the general migration strategy.
    This means that a variant of the streamlined strategy must be used.

.. warning::

    All strategies work for both Docker-based and native installations.
    But for Docker-based installations some peculiarities have to be considered. These peculiarities are handled in the optional steps.

.. note::

    It is also feasible to clone the OTRS datase to the OTOBO database server before the actual migration.
    This can speed up the general migration strategy.

Migration Requirements
----------------------

1.  Basic requirement for a migration is that you already have an ((OTRS)) Community Edition or OTRS 6.0.\* / OTRS 7.0.\* running,
    and that you want to transfer both configuration and data to OTOBO.

.. warning::

    Please consider carefully whether you really need the data and configuration.
    Experience shows that quite often a new start is the better option. This is because in many cases
    the previously used installation and configuration was rather suboptimal anyways.
    It might also make sense to only transfer the ticket data and to change the basic configuration to OTOBO Best Practice.
    We are happy to advise you, please get in touch at hello@otobo.de or ask your question in the OTOBO Community forum at https://forum.otobo.org/.

2.  You need a running OTOBO installation to start the migration from there!

3.  This OTOBO installation must contain all OPM packages installed in your OTRS that you want to use in OTOBO, too.

4.  If you are planning to migrate to another server, then the OTOBO webserver must be able
    to access the location where your ((OTRS)) Community Edition or OTRS 6.0.* / OTRS 7.0.\* is installed.
    In most cases, this is the directory */opt/otrs* on the server running OTRS.
    The read access can be effected via SSH or via file system mounts.

5.  The *otrs* database must be accessible from the server running OTOBO. Readonly access must be granted for external hosts.
    If access is not possible, or when the speed of the migration should be optimised, then a dump of the database is sufficient.

.. note::

    If SSH and database access between the servers is not possible,
    please migrate OTRS to OTOBO on the same server and only then move the new installation.

Step 1: Install the new OTOBO System
------------------------------------

Please start with installing a new OTOBO system. Your old OTRS / ((OTRS)) Community Edition installation will be migrated to that new system.
We strongly recommend to read the chapter :doc:`installation`. For Docker-based installations we refer to the chapter :doc:`installation-docker`.

.. warning::

    Under Apache, there are pitfalls with running two independent *mod_perl* applications under on the same webserver.
    Therefore, it is advised to run OTRS and OTOBO on separate webservers. Alternatively remove the OTRS configuration
    from Apache before installing OTOBO.
    Use the command ``a2query -s`` and check the directories */etc/apache2/sites-available* and */etc/apache2/sites-enabled* for
    inspecting which configurations are currently available and which are enabled.

After finishing the installation please log in as *root@localhost*. Navigate to the OTOBO Admin Area ``Admin -> Packages``
and install all required OTOBO OPM packages.

The following OPM packages and OTRS "Feature Addons" need NOT and should NOT be installed, as these features are already available in the OTOBO standard:
    - OTRSHideShowDynamicField
    - RotherOSSHideShowDynamicField
    - TicketForms
    - RotherOSS-LongEscalationPerformanceBoost
    - Znuny4OTRS-AdvancedDynamicFields
    - Znuny4OTRS-AutoSelect
    - Znuny4OTRS-EscalationSuspend
    - OTRSEscalationSuspend
    - OTRSDynamicFieldDatabase
    - OTRSDynamicFieldWebService
    - OTRSBruteForceAttackProtection
    - Znuny4OTRS-ExternalURLJump
    - Znuny4OTRS-QuickClose
    - Znuny4OTRS-AutoCheckbox
    - OTRSSystemConfigurationHistory
    - Znuny4OTRS-PasswordPolicy

The following OTOBO packages have been integrated into OTOBO 11.0. This means that they should not be installed
in the target system when the target system is OTOBO 11.
    - ImportExport

Step 2: Deactivate ``SecureMode`` on OTOBO
-------------------------------------------------------

After installing OTOBO, please log in again to the OTOBO Admin Area ``Admin -> System Configuration`` and deactivate the config option ``SecureMode``.

.. note::

    Do not forget to actually deploy the changed setting.

Step 3: Stop the OTOBO Daemon
-------------------------------------------------------

This is necessary when the OTOBO Daemon is actually running.
Stopping the Daemon is different between Docker-based and non-Docker-based installations.

In the non-Docker case execute the following commands as the user *otobo*:

.. code-block:: bash

    # in case you are logged in as root
    root> su - otobo

    otobo> /opt/otobo/bin/Cron.sh stop
    otobo> /opt/otobo/bin/otobo.Daemon.pl stop --force

When OTOBO is running in Docker, you just need to stop the service ``daemon``:

.. code-block:: bash

    docker_admin> cd /opt/otobo-docker
    docker_admin> docker-compose stop daemon
    docker_admin> docker-compose ps     # otobo_daemon_1 should have exited with the code 0

.. note::

    It is recommended to run a backup of the whole OTOBO system at this point. If something goes wrong during migration, you will then not have to
    repeat the entire installation process, but can instead import the backup for a new migration.

    .. seealso::

        We advise you to read the OTOBO :doc:`backup-restore` chapter.

Optional Step: Mount /opt/otrs for Convenient Access
----------------------------------------------------

Often OTOBO should be running on a new server where */opt/otrs* isn't available initially.
In these cases the directory */opt/otrs* on the OTRS server can be mounted into the file system
of the OTOBO server. When a regular network mount is not possible, then using ``sshfs`` might be an option.

Optional Step: Install ``sshpass`` and ``rsync`` when */opt/otrs* Should be Copied via ssh
------------------------------------------------------------------------------------------

This step is only necessary when you want to migrate OTRS from another server and when
*/opt/otrs* from the remote server hasn't been mounted on the server running OTOBO.

The tools ``sshpass`` and ``rsync`` are needed so that *migration.pl* can copy files via ssh.
For installing ``sshpass``, please log in on the server as user ``root``
and execute one of the following commands:

.. code-block:: bash

    $ # Install sshpass under Debian / Ubuntu Linux
    $ sudo apt-get install sshpass

.. code-block:: bash

    $ #Install sshpass under RHEL/CentOS Linux
    $ sudo yum install sshpass

.. code-block:: bash

    $ # Install sshpass under Fedora
    $ sudo dnf install sshpass

.. code-block:: bash

    $ # Install sshpass under OpenSUSE Linux
    $ sudo zypper install sshpass

The same thing must be done for *rsync* when it isn't available yet.

Step 4: Preparing the OTRS / ((OTRS)) Community Edition system
----------------------------------------------------------------------------

.. note::

    Be sure to have a valid backup of your OTRS / ((OTRS)) Community Edition system, too. Yes, we do not touch any OTRS data during the migration, but at times
    a wrong entry is enough to cause trouble.

Now we are ready for the migration. First of all we need to make sure that no more tickets are processed and
no users log on to OTRS:

Please log in to the OTRS Admin Area ``Admin ->  System Maintenance`` and add a new system maintenance slot for a few hours.
After that, delete all agent and user sessions (``Admin ->  Sessions``) and log out.

Stop All Relevant Services and the OTRS Daemon
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Please make sure there are no running services or cron jobs.

.. code-block:: bash

    root> su - otrs
    otrs> /opt/otrs/bin/Cron.sh stop
    otrs> /opt/otrs/bin/otrs.Daemon.pl stop --force

Clear the Caches and the Operational Data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The cached data and the operational data doesn't have to be migrated.
The mail queue should at this point already be empty.

.. code-block:: bash

    root> su - otrs
    otrs> /opt/otrs/bin/otrs.Console.pl Maint::Cache::Delete
    otrs> /opt/otrs/bin/otrs.Console.pl Maint::Session::DeleteAll
    otrs> /opt/otrs/bin/otrs.Console.pl Maint::Loader::CacheCleanup
    otrs> /opt/otrs/bin/otrs.Console.pl Maint::WebUploadCache::Cleanup
    otrs> /opt/otrs/bin/otrs.Console.pl Maint::Email::MailQueue --delete-all

Optional Step for Docker: make required data available inside container
------------------------------------------------------------------------

There are some specifics to be considered when your OTOBO installation is running under Docker.
The most relevant: processes running in a Docker container generally cannot access directories
outside the container. There is an exception though: directories mounted as volumes into the container can be accessed.
Also, note that the MariaDB database running in ``otobo_db_1`` is not directly accessible from outside the container network.

.. note::

    In the sample commands, we assume that the user **docker_admin** is used for interacting with Docker.
    The Docker admin may be either the **root** user of the Docker host or a dedicated user with the required permissions.

Copy */opt/otrs* into the volume *otobo_opt_otobo*
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In this section, we assume that the OTRS home directory */opt/otrs* is available
on the Docker host.

There are at least two viable possibilities:

    a. copy */opt/otrs* into the existing volume *otobo_opt_otobo*
    b. mount */opt/otrs* as an additional volume

Let's concentrate on option **a.** here.

First we need to find out where the volume *otobo_opt_otobo* is available on the Docker host.

.. code-block:: bash

    docker_admin> otobo_opt_otobo_mp=$(docker volume inspect --format '{{ .Mountpoint }}' otobo_opt_otobo)
    docker_admin> echo $otobo_opt_otobo_mp  # just a sanity check

For safe copying, we use ``rsync``.
Depending on your Docker setup, the command ``rsync`` might need to be run with ``sudo``.

.. code-block:: bash

    docker_admin> # when docker_admin is root
    docker_admin> rsync --recursive --safe-links --owner --group --chown 1000:1000 --perms --chmod "a-wx,Fu+r,Du+rx" /opt/otrs/ $otobo_opt_otobo_mp/var/tmp/copied_otrs
    docker_admin> ls -la $otobo_opt_otobo_mp/var/tmp/copied_otrs  # just a sanity check

    docker_admin> # when docker_admin is not root
    docker_admin> sudo rsync --recursive --safe-links --owner --group --chown 1000:1000 --perms --chmod "a-wx,Fu+r,Du+rx" /opt/otrs/ $otobo_opt_otobo_mp/var/tmp/copied_otrs
    docker_admin> sudo ls -la $otobo_opt_otobo_mp/var/tmp/copied_otrs  # just a sanity check

This copied directory will be available as */opt/otobo/var/tmp/copied_otrs* within the container.

Step 5: Perform the Migration!
---------------------------------

Please use the web migration tool at http://localhost/otobo/migration.pl. Be aware that you might have to replace "localhost"
with your OTOBO hostname and you might have to add your non-standard port.
The application then guides you through the migration process.

.. warning::

    Sometimes, a warning is shown that the deactivation of **SecureMode** has not been detected.
    Please restart the webserver in this case. This forces the webserver to read in the current configuration.

    .. code-block:: bash

        # native installation
        root> service apache2 restart

        # Docker-based installation
        docker_admin> cd /opt/otobo-docker
        docker_admin> docker-compose restart web
        docker_admin> docker-compose ps     # otobo_web_1 should be running again

.. note::

    If OTOBO runs inside a Docker container, keep the default settings *localhost* for the OTRS server
    and */opt/otobo/var/tmp/copied_otrs* for the OTRS home directory. This is the path of the data that
    was copied in the optional step.

.. note::

    The default values for OTRS database user and password are taken from *Kernel/Config.pm* in the OTRS home directory.
    Change the proposed settings if you are using a dedicated database user for the migration.
    Also change the settings when you work with a database that was copied into the *otobo_db_1* Docker container.

.. note::

    In the Docker case, a database running on the Docker host won't be reachable via ``127.0.0.1`` from within the Docker container.
    This means that the setting ``127.0.0.1`` won't be valid for the input field ``OTRS Server``.
    In that case, enter one of the alternative IP-addresses reported by the command ``hostname --all-ip-addresses`` for ``OTRS Server``.

.. note::

    When migrating to a new application server, or to a Docker-based installation, quite often the database cannot be accessed
    from the target installation. This is usually due to the fact that the otobo database user can only connect from the host the database runs on.
    In order to allow access anyways it is recommended to create a dedicated database user for the migration.
    E.g. ``CREATE USER 'otrs_migration'@'%' IDENTIFIED BY 'otrs_migration';`` and
    ``GRANT SELECT, SHOW VIEW ON otrs.* TO 'otrs_migration'@'%';``.
    This user can be dropped again after the migration: ``DROP USER 'otrs_migration'@'%'``.

Custom settings in *Kernel/Config.pm* are carried over from the old OTRS installation to the new OTOBO installation.
When you have custom settings, then please take a look at the migrated file */opt/otobo/Kernel/Config.pm*.
You might want to adapt custom pathes or LDAP settings. In the best case you might find that some custom setting are longer needed.

When the migration is complete, please take your time and test the entire system. Once you have decided
that the migration was successful and that you want to use OTOBO from now on, start the OTOBO Daemon:

.. code-block:: bash

    root> su - otobo
    otobo>
    otobo> /opt/otobo/bin/Cron.sh start
    otobo> /opt/otobo/bin/otobo.Daemon.pl start

In the Docker case:

.. code-block:: bash

    docker_admin> cd ~/otobo-docker
    docker_admin> docker-compose start daemon

Step 6: After Successful Migration!
------------------------------------

1. Uninstall ``sshpass`` if you do not need it anymore.
2. Drop the databases and database users dedicated to the migration if you created any.
3. Have fun with OTOBO!


Known Migration Problems
-----------------------------------

1. Login after migration not possible
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

During our migration tests, the browser used for the migration sometimes had problems.
After restarting the browser, this problem usually was solved. With Safari it was sometimes necessary to manually delete the old OTRS session.

2. Final page of the migration has a strange layout due to missing CSS files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This can happen when the setting ScriptAlias has a non-standard value. The migration simply substitutes otrs for otobo. This might lead to
the effect that the CSS and JavaScript can no longer be retrieved in OTOBO.
When that happens, please check the settings in *Kernel/Config.pm* and revert them to sane values.

3. Migration stops due to MySQL errors
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On systems that experienced problems with an upgrade in the past, the migration process may stop due to MySQL errors
in the tables *ticket* and *ticket_history*. Usually these errors are NULL values in the source table that are no longer
allowed in the target table. These conflicts have to be manually resolved before you can resume the migration.

There is a check in *migration.pl* that checks for NULL values before the data transfer is done.
Note, that the resolution still needs to be performed manually.

4. Errors in Step 5 when migrating to PostgreSQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In these cases the not so helpful message "System was unable to complete data transfer." is shown by *migration.pl*. The Apache logfile,
and the OTOBO logfile, show a more meaningful message:
"Message: ERROR:  permission denied to set parameter "session_replication_role", SQL: 'set session_replication_role to replica;'".
In order to give the database user **otobo** the needed superuser privileges,
run the following statement as the PostgreSQL admin: ``ALTER USER otobo WITH SUPERUSER;``.
Then retry running http://localhost/otobo/migration.pl.
After the migration, return to the normal state by running ``ALTER USER otobo WITH NOSUPERUSER``.

It is not clear yet, whether the extended privileges have to be granted in every setup.

.. seealso::

    The discussion in https://otobo.de/de/forums/topic/otrs-6-mysql-migration-to-otobo-postgresql/.

5. Problems with the Deployment the Merged System Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The system configuration is migrated after the database tables were migrated. In this context, migration means merging
the default settings of OTOBO with the system configuration of the source OTRS system.
Inconsistencies can arise in this step. An real life example is the setting ``Ticket::Frontend::AgentTicketQuickClose###State``.
This setting is new in OTOBO 10 and the default value is the state ``closed successful``. But this setting is invalid
when the state ``closed successful`` has been dropped or renamed in the source system.
This inconsistency is detected as an error in the migration step **Migrate configuration settings**. Actually,
the merged system configuration is stored in the database, but additional validity checks are performed during deployment.

The problem must be alleviated manually by using OTOBO console commands.

- List the inconsistencies with the command
  ``bin/otobo.Console.pl Admin::Config::ListInvalid``
- Interactively fix the invalid values with
  ``bin/otobo.Console.pl Admin::Config::FixInvalid``
- Deploy the collected changes from migration.pl, including the deactivated **SecureMode** with
  ``bin/otobo.Console.pl Maint::Config::Rebuild``

After these manual steps you should be able to run *migration.pl* again. The migration will continue with the step
where the error occurred.

Step 7: Manual Migration Tasks and Changes
------------------------------------------

1. Password policy rules
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

With OTOBO 10 a new default password policy for agent and customer users is in effect, if local authentication is used. The password policy rules can be changed in the system configuration (``PreferencesGroups###Password`` and ``CustomerPersonalPreference###Password``).

+---------------------------------------+--------------+
| Password Policy Rule                  | Default      |
+=======================================+==============+
| ``PasswordMinSize``                   | 8            |
+---------------------------------------+--------------+
| ``PasswordMin2Lower2UpperCharacters`` | Yes          |
+---------------------------------------+--------------+
| ``PasswordNeedDigit``                 | Yes          |
+---------------------------------------+--------------+
| ``PasswordHistory``                   | 10           |
+---------------------------------------+--------------+
| ``PasswordTTL``                       | 30 days      |
+---------------------------------------+--------------+
| ``PasswordWarnBeforeExpiry``          | 5 days       |
+---------------------------------------+--------------+
| ``PasswordChangeAfterFirstLogin``     | Yes          |
+---------------------------------------+--------------+

2. Under Docker: Manually migrate cron jobs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In a non-Docker installation of OTOBO, there is at least one cron job which checks the health of the Daemon.
Under Docker, this cron job no longer exists.
Furthermore, there is no cron daemon running in any of the Docker containers.
This means that you have to look for an individual solution for OTRS systems with customer-specific cron jobs
(e. g. backing up the database).

Special topics
---------------

Migration from Oracle to Oracle
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For migration to Oracle the ETL-like strategy must be employed.
This is because Oracle provides no easy way to temporarily turn off foreign key checks.

On the OTOBO host a Oracle client and the Perl module ``DBD::Oracle`` must be installed.

.. note::

    When using the Oracle instant client, then the optional SDK is also needed for installing DBD::Oracle.

There are many ways of cloning a schema. In the sample commands we use ``expdb`` and ``impdb`` which use
Data Pump under the hood.

.. note::

    The connect strings shown in this documentation refer to the case when both source and target database
    run in a Docker container. See also https://github.com/bschmalhofer/otobo-ideas/blob/master/oracle.md .


1. Clear out otobo

Stop the webserver for otobo, so that the DB connection for otobo is closed.

.. code-block:: SQL

    -- in the OTOBO database
    DROP USER otobo CASCADE

2. Export the complete OTRS schema.

.. code-block:: bash

   mkdir /tmp/otrs_dump_dir

.. code-block:: SQL

    -- in the OTRS database
    CREATE DIRECTORY OTRS_DUMP_DIR AS '/tmp/otrs_dump_dir';
    GRANT READ, WRITE ON DIRECTORY OTRS_DUMP_DIR TO sys;

.. code-block:: bash

    expdp \"sys/Oradoc_db1@//127.0.0.1/orclpdb1.localdomain as sysdba\" schemas=otrs directory=OTRS_DUMP_DIR dumpfile=otrs.dmp logfile=expdpotrs.log

3. Import the OTRS schema, renaming the schema to 'otobo'.

.. code-block:: bash

    impdp \"sys/Oradoc_db1@//127.0.0.1/orclpdb1.localdomain as sysdba\" directory=OTRS_DUMP_DIR dumpfile=otrs.dmp logfile=impdpotobo.log remap_schema=otrs:otobo

.. code-block:: SQL

    -- in the OTOBO database
    -- double check
    select owner, table_name from all_tables where table_name like 'ARTICLE_DATA_OT%_CHAT';

    -- optionally, set the password for the user otobo
        ALTER USER otobo IDENTIFIED BY XXXXXX;

4. Adapt the cloned schema otobo

.. code-block:: bash

    cd /opt/otobo
    scripts/backup.pl --backup-type migratefromotrs # it's OK that the command knows only about the otobo database, only last line is relevant
    sqlplus otobo/otobo@//127.0.0.1/orclpdb1.localdomain < /home/bernhard/devel/OTOBO/otobo/2021-03-31_13-36-55/orclpdb1.localdomain_post.sql >sqlplus.out 2>&1
    double check with `select owner, table_name from all_tables where table_name like 'ARTICLE_DATA_OT%_CHAT';

5. Start the web server for otobo again

6. Proceed with step 5, that is with running ``migration.pl``.

.. note::

    If migrating to OTOBO version greater or equal 10.1 the script ``/opt/otobo/scripts/DBUpdate-to-10.1.pl`` has to be executed, to create the tables ``stats_report`` & ``data_storage``, which were newly added in version 10.1.
    

Optional Step: Streamlined migration of the database (only for experts and spezial scenarios)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the general migration strategy, all data in the database tables is copied row by row from the OTRS database
into the OTOBO database.
Exporting the data from the OTRS database and importing it into the OTOBO database might save time and is more
stable in some circumstances.

.. note::

    This variant works for both Docker-based and native installations.

.. note::

    These instructions assume that OTRS is using MySQL as its backend.

First of all, we need a dump of the needed OTRS database tables.
Then we need to perform a couple of transformations:

    - convert the character set to *utf8mb4*
    - rename a couple of tables
    - shorten some table columns

After the transfomation we can overwrite the tables in the OTOBO schema with the transformed data from OTRS.
Effectively we need not a single dump file, but several SQL scripts.

When ``mysqldump`` is installed and a connection to the OTRS database is possible,
you can create the database dump directly on the Docker host. This case is supported
by the script *bin/backup.pl*.

.. warning::

    This requires that an OTOBO installation is available on the Docker host.

.. code-block:: bash

    otobo> cd /opt/otobo
    otobo> scripts/backup.pl -t migratefromotrs --db-name otrs --db-host=127.0.0.1 --db-user otrs --db-password "secret_otrs_password"

.. note::

    Alternatively, the database can be dumped on another server and then be transferred to the Docker host afterwards.
    An easy way to do this is to copy */opt/otobo* to the server running OTRS and perform the same command as above.

The script *bin/backup.pl* generates four SQL scripts in a dump directory, e.g. in *2021-04-13_12-13-04*
In order to execute the SQL scripts, we need to run the command ``mysql``.

Native installation:

.. code-block:: bash

    otobo> cd <dump_dir>
    otobo> mysql -u root -p<root_secret> otobo < otrs_pre.sql
    otobo> mysql -u root -p<root_secret> otobo < otrs_schema_for_otobo.sql
    otobo> mysql -u root -p<root_secret> otobo < otrs_data.sql
    otobo> mysql -u root -p<root_secret> otobo < otrs_post.sql

Docker-based installation:

Run the command ``mysql`` within the Docker container *db* for importing the database dump files.
Note that the password for the database root is now the password
that has been set up in the file *.env* on the Docker host.

.. code-block:: bash

    docker_admin> cd /opt/otobo-docker
    docker_admin> docker-compose exec -T db mysql -u root -p<root_secret> otobo < /opt/otobo/<dump_dir>/otrs_pre.sql
    docker_admin> docker-compose exec -T db mysql -u root -p<root_secret> otobo < /opt/otobo/<dump_dir>/otrs_schema_for_otobo.sql
    docker_admin> docker-compose exec -T db mysql -u root -p<root_secret> otobo < /opt/otobo/<dump_dir>/otrs_data.sql
    docker_admin> docker-compose exec -T db mysql -u root -p<root_secret> otobo < /opt/otobo/<dump_dir>/otrs_post.sql

For a quick check whether the import worked, you can run the following commands.

.. code-block:: bash

    otobo> mysql -u root -p<root_secret> -e 'SHOW DATABASES'
    otobo> mysql -u root -p<root_secret> otobo -e 'SHOW TABLES'
    otobo> mysql -u root -p<root_secret> otobo -e 'SHOW CREATE TABLE ticket'

or when running under Docker

.. code-block:: bash

    docker_admin> docker-compose exec -T db mysql -u root -p<root_secret> -e 'SHOW DATABASES'
    docker_admin> docker-compose exec -T db mysql -u root -p<root_secret> otobo -e 'SHOW TABLES'
    docker_admin> docker-compose exec -T db mysql -u root -p<root_secret> otobo -e 'SHOW CREATE TABLE ticket'

The database is now migrated. This means that during the next step we can skip the database migration.
Watch out for the relevant checkbox.
