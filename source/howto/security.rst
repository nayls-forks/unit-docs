##################
Security Checklist
##################

At its core, Unit has security as one of its top priorities; our development
follows the appropriate best practices focused on making the code robust and
solid.  However, even the most hardened system requires proper setup,
configuration, and maintenance.

This guide lists the steps to protect your Unit from installation to individual
app configuration.


.. _security-update:

*********************
Update Unit Regularly
*********************

**Rationale**: Each release introduces `bug fixes and new
features </CHANGES.txt>`_ that improve your installation's security.

**Actions**: Follow our latest `news
<https://mailman.nginx.org/mailman/listinfo/unit>`_ and upgrade to new
versions shortly after they are released.

.. nxt_details:: Details

   Specific upgrade steps depend on your installation method:

   - The recommended option is to use our official :ref:`packages
     <installation-precomp-pkgs>` or Docker :ref:`images
     <installation-docker>`; with them, it's just a matter of updating
     :samp:`unit-*` packages with your package manager of choice or
     switching to a newer image.

   - If you use a third-party installation :ref:`method
     <installation-community-repos>`, consult the maintainer's documentation
     for details.

   - If you install Unit from :ref:`source files <installation-src>`,
     rebuild and reinstall Unit and its modules from scratch.


.. _security-socket-state:

***********************
Secure Socket and State
***********************

**Rationale**: Your :ref:`control socket and state directory
<installation-src-dir>` provide unlimited access to Unit's configuration, which
calls for stringent protection.

**Actions**: Default configuration in our :ref:`official packages
<installation-precomp-pkgs>` is usually sufficient; if you use another
installation method, ensure the control socket and the state directory are
safe.

.. nxt_details:: Control Socket

   If you use a Unix
   control socket, ensure it is available to :samp:`root` only:

   .. code-block:: console

      $ unitd -h

            ...
            --control ADDRESS    set address of control API socket
                                 default: "unix:/path/to/control.unit.sock"

      # ls -l /path/to/control.unit.sock

            srw------- 1 root root 0 ... /path/to/control.unit.sock

   Unix domain sockets aren't network accessible; for remote access, use
   :ref:`NGINX <nginx-secure-api>` or a solution such as SSH:

   .. code-block:: console

      $ ssh -N -L :nxt_hint:`./here.sock <Local socket>`::nxt_ph:`/path/to/control.unit.sock <Socket on the Unit server>` root@:nxt_hint:`unit.example.com <Unit server hostname>` &
      $ curl --unix-socket :nxt_hint:`./here.sock <Use local socket to configure Unit>`

            {
                "certificates": {},
                "config": {
                    "listeners": {},
                    "applications": {}
                }
            }

   If you prefer an IP-based control socket, avoid public IPs; they expose the
   :ref:`control API <configuration-mgmt>` and all its capabilities.  This
   means your Unit instance can be manipulated by whoever is physically able to
   connect:

   .. code-block:: console

      # unitd --control 10.1.1.14:8080
      $ curl 10.1.1.14:8080

            {
                "certificates": {},
                "config": {
                    "listeners": {},
                    "applications": {}
                }
            }

   Instead, opt for the loopback address to ensure all access is local to your
   server:

   .. code-block:: console

      # unitd --control 127.0.0.1:8080
      $ curl 10.1.1.14:8080

          curl: (7) Failed to connect to 10.1.1.14 port 8080: Connection refused

   However, any processes local to the same system can access the local socket,
   which calls for additional measures.  A go-to solution would be using NGINX
   to :ref:`proxy <nginx-secure-api>` Unit's control API.


.. nxt_details:: State Directory

   The state directory stores Unit's internal configuration between launches.
   Avoid manipulating it or relying on its contents even if tempted to do so.
   Instead, use only the control API to manage Unit's configuration.

   Also, the state directory should be available only to :samp:`root` (or the
   user that the :samp:`main` :ref:`process <security-apps>` runs as):

   .. code-block:: console

      $ unitd -h

            ...
            --state DIRECTORY    set state directory name
                                 default: "/path/to/unit/state/"

      # ls -l /path/to/unit/state/

            drwx------ 2 root root 4096 ...


.. _security-ssl:

*****************
Configure SSL/TLS
*****************

**Rationale**: To protect your client connections in production scenarios,
configure SSL certificate bundles for your Unit installation.

**Actions**: For details, see :ref:`configuration-ssl` and :doc:`certbot`.


.. _security-routes:

***********************
Error-Proof Your Routes
***********************

**Rationale**: Arguably, :ref:`routes <configuration-routes>` are the most
flexible and versatile part of the Unit configuration.  Thus, they must be as
clear and robust as possible to avoid loose ends and gaping holes.

**Actions**: Familiarize yourself with the :ref:`matching
<configuration-routes-matching>` logic and double-check all :ref:`patterns
<configuration-routes-matching-patterns>` that you use.

.. nxt_details:: Details

   Some considerations:

   - Mind that :ref:`variables <configuration-variables>` contain arbitrary
     user-supplied request values; variable-based :samp:`pass` values in
     :ref:`listeners <configuration-listeners>` and :ref:`routes
     <configuration-routes-action>` must account for malicious requests, or the
     requests must be properly filtered.

   - Create :ref:`matching rules <configuration-routes-matching>` to
     formalize the restrictions of your Unit instance and the apps it runs.

   - Configure :ref:`shares <configuration-static>` only for directories and
     files you intend to make public.


.. _security-apps:

****************
Protect App Data
****************

**Rationale**: Unit's architecture involves many processes that operate
together during app delivery; improper process permissions can make sensitive
files available across apps or even publicly.

**Actions**: Properly configure your app directories and shares: apps and the
router process need access to them.  Still, avoid loose rights such as the
notorious :samp:`777`, instead assigning them on a need-to-know basis.

.. nxt_details:: File Permissions

   To configure file permissions for your apps, check Unit's build-time and
   run-time options first:

   .. subs-code-block:: console

      $ unitd -h

            --user USER          set non-privileged processes to run as specified user
                                 default: ":nxt_hint:`unit_user <Build-time setting, can be overridden>`"

            --group GROUP        set non-privileged processes to run as specified group
                                 default: user's primary group

      $ ps ax | grep unitd

            ... unit: main v|version| [... --user :nxt_hint:`unit_user <Make sure to check for runtime overrides>` --group unit_group ...]

   In particular, this is the account the router process runs as.  Use this
   information to set up permissions for the app code or binaries and shared
   static files.  The main idea is to limit each app to its own files and
   directories while simultaneously allowing Unit's router process to access
   static files for all apps.

   Specifically, the requirements are as follows:

   - All apps should run as different users so that the permissions can be
     configured properly.  Even if you run a single app, it's reasonable to
     create a dedicated user for added flexibility.

   - An app's code or binaries should be reachable for the user the app runs
     as; the static files should be reachable for the router process.  Thus,
     each part of an app's directory path must have execute permissions
     assigned for the respective users.

   - An app's directories should not be available to other apps or
     non-privileged system users. The router process should be able to access
     the app's static file directories.  Accordingly, the app's directories
     must have read and execute permissions assigned for the respective users.

   - The files and directories that the app is designed to update should
     be writable only for the user the app runs as.

   - The app code should be readable (and executable in case of :ref:`external
     <modules-ext>` apps) for the user the app runs as; the static content
     should be readable for the router process.

   A detailed walkthrough to guide you through each requirement:

   #. If you have several independent apps, running them with a single user
      account poses a security risk.  Consider adding a separate system user
      and group per each app:

      .. code-block:: console

         # :nxt_hint:`useradd <Add user account without home directory>` -M app_user
         # groupadd app_group
         # :nxt_hint:`usermod <Deny interactive login>` -L app_user
         # :nxt_hint:`usermod <Add user to the group>` -a -G app_group app_user

      Even if you run a single app, this helps if you add more apps or need to
      decouple permissions later.

   #. It's important to add Unit's non-privileged user account to *each* app
      group:

      .. code-block:: console

         # usermod -a -G app_group unit_user

      Thus, Unit's router process can access each app's directory and serve
      files from each app's shares.

   #. A frequent source of issues is the lack of permissions for directories
      inside a directory path needed to run the app, so check for that if in
      doubt.  Assuming your app code is stored at :samp:`/path/to/app/`:

      .. code-block:: console

         # ls -l /

               :nxt_hint:`drwxr-xr-x <Permissions are OK>`  some_user some_group  path

         # ls -l /path/

               :nxt_hint:`drwxr-x--- <Permissions are too restrictive>`  some_user some_group  to

      This may be a problem because the :samp:`to/` directory isn't owned by
      :samp:`app_user:app_group` and denies all permissions to non-owners (as
      the :samp:`---` sequence tells us), so a fix can be warranted:

      .. code-block:: console

         # :nxt_hint:`chmod <Add read/execute permissions for non-owners>` o+rx /path/to/

      Another solution is to add :samp:`app_user` to :samp:`some_group`
      (assuming this was not done before):

      .. code-block:: console

         # usermod -a -G some_group app_user

   #. Having checked the directory tree, assign ownership and permissions for
      your app's directories, making them reachable for Unit and the app:

      .. code-block:: console

         # :nxt_hint:`chown <Assign ownership for the app code>` -R app_user:app_group /path/to/app/
         # :nxt_hint:`chown <Assign ownership for the static files>` -R app_user:app_group /path/to/static/app/files/
         # find /path/to/app/ -type d -exec :nxt_hint:`chmod <Add read/execute permissions to app code directories for user and group>` u=rx,g=rx,o= {} \;
         # find /path/to/static/app/files/ -type d -exec :nxt_hint:`chmod <Add read/execute permissions to static file directories for user and group>` u=rx,g=rx,o= {} \;

   #. If the app needs to update specific directories or files, make sure
      they're writable for the app alone:

      .. code-block:: console

         # :nxt_hint:`chmod <Add write permissions for the user only; the group shouldn't have them>` u+w /path/to/writable/file/or/directory/

      In case of a writable directory, you may also want to prevent non-owners
      from messing with its files:

      .. code-block:: console

         # :nxt_hint:`chmod <Sticky bit prevents non-owners from deleting or renaming files>` +t /path/to/writable/directory/

      .. note::

         Usually, apps store and update their data outside the app code
         directories, but some apps may mix code and data.  In such a case,
         assign permissions on an individual basis, making sure you understand
         how the app uses each file or directory: is it code, read-only
         content, or writable data.

   #. For :ref:`embedded <modules-emb>` apps, it's usually enough to make the
      app code and the static files readable:

      .. code-block:: console

         # find /path/to/app/code/ -type f -exec :nxt_hint:`chmod <Add read rights to app code for user and group>` u=r,g=r,o= {} \;
         # find /path/to/static/app/files/ -type f -exec :nxt_hint:`chmod <Add read rights to static files for user and group>` u=r,g=r,o= {} \;

   #. For :ref:`external <modules-emb>` apps, additionally make the app code or
      binaries executable:

      .. code-block:: console

         # find /path/to/app/ -type f -exec :nxt_hint:`chmod <Add read and execute rights to app code for user and group>` u=rx,g=rx,o= {} \;
         # find /path/to/static/app/files/ -type f -exec :nxt_hint:`chmod <Add read rights to static files for user and group>` u=r,g=r,o= {} \;

   #. To run a single app, :doc:`configure <../configuration>` Unit as follows:

      .. code-block:: json

         {
             "listeners": {
                 ":nxt_hint:`*:80 <Or another suitable socket address>`": {
                     "pass": "routes"
                 }
             },

             "routes": [
                 {
                     "action": {
                         "share": ":nxt_ph:`/path/to/static/app/files/ <Router process needs read and execute permissions to serve static content from this directory>`",
                         "fallback": {
                             "pass": "applications/app"
                         }
                     }
                 }
             ],

             "applications": {
                 "app": {
                     "type": "...",
                     "user": "app_user",
                     "group": "app_group"
                 }
             }
         }

   #. To run several apps side by side, :doc:`configure <../configuration>`
      them with appropriate user and group names.  The following
      configuration distinguishes apps based on the request URI, but you can
      implement another scheme such as different listeners:

      .. code-block:: json

         {
             "listeners": {
                 ":nxt_hint:`*:80 <Or another suitable socket address>`": {
                     "pass": "routes"
                 }
             },

             "routes": [
                 {
                     "match": {
                         "uri": "/app1/*"
                     },

                     "action": {
                         "share": ":nxt_ph:`/path/to/static/app1/files/ <Router process needs read and execute permissions to serve static content from this directory>`",
                         "fallback": {
                             "pass": "applications/app1"
                         }
                     }
                 },

                 {
                     "match": {
                         "uri": "/app2/*"
                     },

                     "action": {
                         "share": ":nxt_ph:`/path/to/static/app2/files/ <Router process needs read and execute permissions to serve static content from this directory>`",
                         "fallback": {
                             "pass": "applications/app2"
                         }
                     }
                 }
             ],

             "applications": {
                 "app1": {
                     "type": "...",
                     "user": "app_user1",
                     "group": "app_group1"
                 },

                 "app2": {
                     "type": "...",
                     "user": "app_user2",
                     "group": "app_group2"
                 }
             }
         }

   .. note::

      As usual with permissions, different steps may be required if you use
      ACLs.

.. nxt_details:: App Internals

   Unfortunately, quite a few web apps are built in a manner that mixes their
   source code, data, and configuration files with static content, which calls
   for complex access restrictions.  The situation is further aggravated by the
   inevitable need for maintenance activities that may leave a footprint of
   extra files and directories unrelated to the app's operation.  The issue has
   several aspects:

   - Storage of code and data at the same locations, which usually happens by
     (insufficient) design.  You neither want your internal data and code files
     to be freely downloadable nor your user-uploaded data to be executable as
     code, so configure your routes and apps to prevent both.

   - Exposure of configuration data.  Your app-specific settings, :file:`.ini`
     or :file:`.htaccess` files, and credentials are best kept hidden from
     prying eyes, and your routing configuration should reflect that.

   - Presence of hidden files from versioning, backups by text editors, and
     other temporary files.  Instead of carving your configuration around
     these, it's best to keep your app free of them altogether.

   If these can't be avoided, investigate the inner workings of the app to
   prevent exposure, for example:

   .. code-block:: json

         {
             "routes": {
                 "app": [
                     {
                         "match": {
                             ":nxt_hint:`uri <Handles requests that target PHP scripts to avoid having them served as static files>`": [
                                 "*.php",
                                 "*.php/*"
                             ]
                         },

                         "action": {
                             "pass": "applications/app/direct"
                         }
                     },
                     {
                         "match": {
                             ":nxt_hint:`uri <Protects files and directories best kept hidden>`": [
                                 ":nxt_hint:`!/sensitive/* <Restricts access to a directory with sensitive data>`",
                                 ":nxt_hint:`!/data/* <Restricts access to a directory with sensitive data>`",
                                 ":nxt_hint:`!/app_config_values.ini <Restricts access to a specific file>`",
                                 ":nxt_hint:`!*/.* <Restricts access to hidden files and directories>`",
                                 ":nxt_hint:`!*~ <Restricts access to temporary files>`"
                             ]
                         },

                         "action": {
                             ":nxt_hint:`share <Serves valid requests with static content>`": "/path/to/app/",
                             ":nxt_hint:`types <Limits file types served from the share>`": [
                                 "image/*",
                                 "text/*",
                                 "application/javascript"
                             ],

                             ":nxt_hint:`fallback <Relays all requests not yet served to a catch-all app target>`": {
                                 "pass": "applications/app/index"
                             }
                         }
                     }
                 ]
             }
         }

   However, this does not replace the need to set up file permissions; use both
   :ref:`matching rules <configuration-routes-matching>` and per-app user
   permissions to manage access.  For more info and real-life examples, refer
   to our app :doc:`howtos <index>` and the 'File Permissions' callout above.

.. nxt_details:: Unit's Process Summary

   .. _security-processes:

   Unit's processes are detailed `elsewhere
   <https://www.nginx.com/blog/introducing-nginx-unit/>`_, but here's a
   synopsis of the different roles they have:

   .. list-table::
      :header-rows: 1

      * - Process
        - Privileged?
        - User and Group
        - Description

      * - Main
        - Yes
        - Whoever starts the :file:`unitd` executable; by default,
          :samp:`root`.
        - Runs as a daemon, spawning Unit's non-privileged and app processes;
          requires numerous system capabilities and privileges for operation.

      * - Controller
        - No
        - Set by :option:`!--user` and :option:`!--group` options at
          :ref:`build <installation-config-src>` or :ref:`execution
          <installation-src-startup>`; by default, :samp:`unit`.
        - Serves the control API, accepting reconfiguration requests,
          sanitizing them, and passing them to other processes for
          implementation.

      * - Discovery
        - No
        - Set by :option:`!--user` and :option:`!--group` options at
          :ref:`build <installation-config-src>` or :ref:`execution
          <installation-src-startup>`; by default, :samp:`unit`.
        - Discovers the language modules in the module directory at startup,
          then quits.

      * - Router
        - No
        - Set by :option:`!--user` and :option:`!--group` options at
          :ref:`build <installation-config-src>` or :ref:`execution
          <installation-src-startup>`; by default, :samp:`unit`.
        - Serves client requests, accepting them, processing them on the spot,
          passing them to app processes, or proxying them further; requires
          access to static content paths you configure.

      * - App processes
        - No
        - Set by per-app :samp:`user` and :samp:`group`
          :ref:`options <configuration-applications>`; by default,
          :option:`!--user` and :option:`!--group` values.
        - Serve client requests that are routed to apps; require access to
          paths and namespaces you configure for the app.

   You can check all of the above on your system when Unit is running:

   .. subs-code-block:: console

      $ ps aux | grep unit

            ...
            root   ... unit: main v|version|
            unit   ... unit: controller
            unit   ... unit: router
            unit   ... unit: "foobar" application

   The important outtake here is to understand that Unit's non-privileged
   processes don't require running as :samp:`root`.  Instead, they should have
   the minimal privileges required to operate, which so far means the ability
   to open connections and access the application code and the static files
   shared during routing.


.. _security-logs:

***************************
Prune Debug and Access Logs
***************************

**Rationale**: Unit stores potentially sensitive data in its general and access
logs; their size can also become a concern if debug mode is enabled.

**Actions**: Secure access to the logs and ensure they don't exceed the allowed
disk space.

.. nxt_details:: Details

   Unit can maintain two different logs:

   - A general-purpose log that is enabled by default and can be switched to
     debug mode for verbosity.

   - An access log that is off by default but can be enabled via the control
     API.

   If you enable debug-mode or access logging, rotate these logs with tools
   such as :program:`logrotate` to avoid overgrowth.  A sample
   :program:`logrotate` `configuration
   <https://man7.org/linux/man-pages/man8/logrotate.8.html#CONFIGURATION_FILE_DIRECTIVES>`_
   (use real log and PID file paths):

   .. code-block:: none

      :nxt_ph:`/path/to/unit.log <Use a real path>` {
          daily
          missingok
          rotate 7
          compress
          delaycompress
          nocreate
          notifempty
          su root root
          postrotate
              if [ -f :nxt_ph:`/path/to/unit.pid <Use a real path>` ]; then
                  :nxt_hint:`/bin/kill <Signals Unit to reopen the log>` -SIGUSR1 `cat /path/to/unit.pid`
              fi
          endscript
      }

   Another issue is the logs' accessibility.  Logs are opened and updated by
   the :ref:`main process <security-apps>` that usually runs as :samp:`root`.
   However, to make them available for a certain consumer, you may need to
   enable access for a dedicated user that the consumer runs as.

   Perhaps, the most straightforward way to achieve this is to assign log
   ownership to the consumer's account.  Suppose you have a log utility running
   as :samp:`log_user:log_group`:

   .. subs-code-block:: console

      $ :nxt_hint:`unitd <Check where the general-purpose log is>` --help

            ...
            --log FILE           set log filename
                                 default: "/default/path/to/unit.log"

      $ :nxt_hint:`ps ax <Make sure the default log path is not overridden at startup>` | grep unitd

            ... unit: main v|version| [... --log :nxt_ph:`/path/to/unit.log <If it's overridden, use the runtime setting>` ...]

      # :nxt_hint:`chown <Assign ownership to the log user>` log_user:log_group /path/to/unit.log

      # :nxt_hint:`curl <Enable access log in the Unit configuration at the given path>` -X PUT -d '"/path/to/access.log"'  \
             --unix-socket /path/to/control.unit.sock \
             http://localhost/config/access_log

            {
                "success": "Reconfiguration done."
            }

      # :nxt_hint:`chown <Assign ownership to the log user>` log_user:log_group /path/to/access.log

   If you change the log file ownership, adjust your :program:`logrotate`
   settings accordingly:

   .. code-block:: none

      /path/to/unit.log {
          ...
          su log_user log_group
          ...
      }

   .. note::

      As usual with permissions, different steps may be required if you use
      ACLs.


.. _security-isolation:

***************************
Add Restrictions, Isolation
***************************

**Rationale**: If the underlying OS allows, Unit provides features that create an
additional level of separation and containment for your apps, such as:

- Share :ref:`path restrictions <configuration-share-path>`
- Namespace and file system root :ref:`isolation
  <configuration-proc-mgmt-isolation>`

**Actions**: For more details, see our blog posts on `path restrictions
<https://www.nginx.com/blog/nginx-unit-updates-for-summer-2021-now-available/#Static-Content:-Chrooting-and-Path-Restrictions>`__,
`namespace <https://www.nginx.com/blog/application-isolation-nginx-unit/>`_ and
`file system <https://www.nginx.com/blog/filesystem-isolation-nginx-unit/>`_
isolation.
