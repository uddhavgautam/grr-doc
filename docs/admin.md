This document is intended for administrators who wish to deploy and
manage GRR in an enterprise. It is also applicable for anyone who wants
to try it out.

# Deploying the GRR server.

Get a GRR server installed and running in a few minutes with the
[quickstart install](quickstart.md). If you’re building a dev server
take a look at the [pip install
instructions](https://github.com/google/grr-doc/blob/markdown/installfrompip.md).

# Client and Server Version Compatibility and Numbering

We try hard to avoid breaking backwards compatibility for clients since
upgrading can be painful, but occasionally we need to make changes that
require a new client version to work. As a general rule you want to
upgrade the server first, then upgrade the clients fairly soon after.

Matching major/minor versions of client and server should work well
together. i.e. Clients 3.1.0.0 and 3.1.6.2 should work well with servers
3.1.0.0 and 3.1.9.7 because they are all 3.1 series. We introduced this
approach for the 3.1.0.0 release.

For older servers and clients, matching the last digit provided similar
guarantees. i.e. client 3.0.0.7 was released with server 0.3.0-7 and
should work well together.

# Key Management

GRR requires multiple key pairs. These are used to:

  - Sign the client certificates for enrollment.

  - Sign and decrypt messages from the client.

  - Sign code and drivers sent to the client.

These keys can be generated using the config\_updater script normally
installed in the path as grr\_config\_updater using the generate\_keys
command.

``` shell
db@host:$ sudo grr_config_updater generate_keys
Generating executable signing key
..............+++
.....+++
Generating driver signing key
..................................................................+++
.............................................................+++
Generating CA keys
Generating Server keys
Generating Django Secret key (used for xsrf protection etc)
db@host:$
```

To Encrypt and add a password to the code & driver signing certificates

1.  Copy the keys (PrivateKeys.executable\_signing\_private\_key &
    PrivateKeys.driver\_signing\_private\_key) from the current GRR
    configuration file (whichever you’re using)
    
      - /etc/grr/grr-server.yaml
    
      - /etc/grr/server.local.yaml

2.  Save these two keys as new text files temporarily. You’ll need to
    convert the key text to the normal format by removing the leading
    whitespace and blank
        lines:
    
        cat <original.exe.private.key.txt> | sed 's/^  //g' | sed '/^$/d' > <clean.exe.private.key>
        cat <original.driver.private.key.txt> | sed 's/^  //g' | sed '/^$/d' > <clean.driver.private.key>

3.  Encrypt key & add
        password
    
        openssl rsa -des3 -in <clean.exe.private.key.txt> -out <exe.private.secure.key>
        openssl rsa -des3 -in <clean.driver.private.key.txt> -out <driver.private.secure.key>

4.  Securely wipe all temporary files with cleartext keys.

5.  Replace the keys in the GRR config with the new encrypted keys (or
    store them offline). Ensure the server is restarted to load the
    updated
configuration.

<!-- end list -->

``` 
 PrivateKeys.executable_signing_private_key: '-----BEGIN RSA PRIVATE KEY-----

   Proc-Type: 4,ENCRYPTED

   DEK-Info: DES-EDE3-CBC,8EDA740783B7563C


   <start key text after *two* blank lines…>

   <KEY...>

   -----END RSA PRIVATE KEY-----'
```

> **Note**
> 
> In the YAML encoding,there **must** be an extra line between the
> encrypted PEM header and the encoded key. The key is double-spaces and
> indented two spaced exactly like all other keys in configuration file.

Alternatively, you can also keep your new, protected keys in files on
the server and load them in the configuration using the file filter like
this:

    PrivateKeys.executable_signing_private_key: %(<path_to_keyfile>|file)

# User Management

GRR has a concept of users of the system. The GUI supports
authentication and this verfication of user identity is used in all
auditing functions (So for example GRR can properly record which user
accessed which client, and who executed flows on clients).

Users are modeled in the data store as AFF4 objects called GRRUser.
These normally reside in the directory *aff4:/users/\<username\>*. To
manage users it is possible to use the config\_updater.py script:

To add the user joe as an admin:

``` shell
db@host:~$ sudo grr_config_updater add_user joe
Using configuration <ConfigFileParser filename="/etc/grr/grr-server.conf">
Please enter password for user 'joe':
Updating user joe

Username: joe
Labels:
Password: set
```

To list all users:

``` shell
db@host:~$ sudo grr_config_updater show_user
Using configuration <ConfigFileParser filename="/etc/grr/grr-server.conf">

Username: test
Labels:
Password: set

Username: admin
Labels: admin
Password: set
```

To update a user (useful for setting labels or for changing
    passwords):

    db@host:~$ sudo grr_config_updater update_user joe --add_labels admin,user
    Using configuration <ConfigFileParser filename="/etc/grr/grr-server.conf">
    Updating user joe
    
    Username: joe
    Labels: admin,user
    Password: set

## Authentication to the Admin UI.

The AdminUI uses basic authentication by default, based on the passwords
within the user objects stored in the data store, but we *don’t expect
you to use this in production*. There is so much diversity and
customization in enterprise authentication shemes that there isn’t a
good way to provide a solution that works for a majority of users. But
you probably already have internal webapps that use authentication, this
is just one more. Most people have found the easiest approach is to sit
Apache (or similar) in front of the GRR Admin UI as a reverse proxy and
use an existing SSO plugin that already works for that platform.
Alternatively, with more work you can handle auth inside GRR by writing
a Webauth Manager (AdminUI.webauth\_manager config option) that uses an
SSO or SAML based authentication mechanism.

# Email Configuration

This section assumes you have already installed an MTA, such as
[Postfix](http://www.postfix.org/) or
[nullmailer](http://untroubled.org/nullmailer/). After you have
successfully tested your mail transfer agent, please proceed to the
steps outlined below.

To configure GRR to send emails for reports or other purposes:

Ensure email settings are correct by running back through the
configuration script if needed (or by checking
/etc/grr/server.local.yaml):

    grr_config_updater initialize

Edit /etc/grr/server.local.yaml to include the following at the end of
the file:

    Worker.smtp_server: <server>
    Worker.smpt_port: <port>

and, if needed,

    Worker.smtp_starttls: True
    Worker.smtp_user: <user>
    Worker.smtp_password: <password>

After configuration is complete, restart the GRR worker(s). You can test
this configuration by running a ClientListReport Flow (Start Global
Flows \> Reporting \> RunReport).

# Security Considerations

Because GRR is designed to be deployed on the Internet and provides very
valuable functionality to an attacker, it comes with a number of
security considerations to think about before deployment. This section
will cover the key security mechanisms and the options you have.

## Communication Security.

GRR communication happens using signed and encrypted protobuf messages.
We use 1024 bit RSA keys to protect symmetric AES128 encryption. The
security of the system does not rely on SSL transport for communication
security. This enables easy replacement of the comms protocol with
non-http mechanisms such as UDP packets.

The communications use a CA and server public key pair generated on
server install. The CA public key is deployed to the client so that it
can ensure it is communicating with the correct server. If these keys
are not kept secure, anyone with MITM capability can intercept
communications and take control of your clients. Additionally, if you
lose these keys, you lose the ability to communicate with your clients.

Full details of this protocol and the security properties can be found
in the [Implementation document](implementation.md).

## Driver, Code Signing and CA Keys.

In addition to the CA and Server key pairs, GRR maintains a set of code
signing and driver signing keys. By default GRR aims to provide only
read-only actions, this means that GRR is unlikely to modify evidence,
and cannot trivially be used to take control of systems running the
agent \[1\]. However, there are a number of use cases where it makes
sense to have GRR execute arbitrary code as explained in the section
[Deploying Custom Drivers and Code](#deploying-custom-drivers-and-code).

As part of the GRR design, we decided that administrative control of the
GRR server shouldn’t trivially lead to code execution on the clients. As
such we embed a strict [whitelist of
commands](https://github.com/google/grr/search?q=IsExecutionWhitelisted)
that can be executed on the client and we have a separate set of keys
for driver signing and code signing. For a driver to be loaded, or
binary to be run the code has to be signed by the specific key, the
client will confirm this signature before execution.

This mechanism helps give the separation of control required in some
deployments. For example, the Incident Response team need to analyze
hosts to get their job done, but deployment of new code to the platfrom
is only done when blessed by the administrators and rolled out as part
of standard change control. The signing mechanism allows Incident
Response to react fast with new code if necessary, but only with the
blessing of the Signing Key held by the platform administrator.

In the default install, the driver and code signing private keys are not
passphrase protected. In a secure environment we strongly recommended
generating and storing these keys off the GRR server and doing offline
signing every time this functionality is required, or at a minimum
setting passphrases which are required on every use. We recommend
encrypting the keys in the config with PEM encryption, config\_updater
will then ask for the passphrase when they are used. An alternative is
to keep a separate offline config that contains the private keys.

## Agent Protection.

The open source agent does not contain protection against being disabled
by administrator/root on the machine. E.g. on Windows, if an attacker
stops the service, the agent will stop and will no longer be reachable.
Currently, it is up to the deployer of GRR to provide more protection
for the service.

## Obfuscation.

If every deployment in the world is running from the same location and
the same code, e.g. c:\\program files\\grr\\grr.exe, it becomes a pretty
obvious thing for an attacker to look for and disable. Luckily the
attacker has the same problem an investigator has in finding malware on
a system, and we can use the same techniques to protect the client. One
of the key benefits of having an open architecture is that customization
of the client and server is easy, and completely within your control.

For a test, or low security deployment, using the defaults open source
agents is fine. However, in a secure environment we strongly recommend
using some form of obfuscation.

This can come in many forms, but to give some examples:

  - Changing service, and binary names

  - Changing registry keys

  - Obfuscating the underlying python code

  - Using a packer to obfuscate the resulting binary

  - Implementing advanced protective or obfuscation functionality such
    as those used in rootkits

  - Implementing watchers to monitor for failure of the client

GRR does not include any obfuscation mechanisms by default. But we
attempt to make this relatively easy by controlling the build process
through the configuration file.

## Enrollment.

In the default setup, clients can register to the GRR server with no
prior knowledge. This means that anyone who has a copy of the GRR agent,
and knows the address of your GRR server can register their client to
your deployment. This significantly eases deployment, and is generally
considered low risk as the client has no control or trust on the server.

However, it does introduce some risk, in particular:

  - If there are flows or hunts you deploy to the entire fleet, a
    malicious client may receive them. These could give away information
    about what you are searching for.

  - Clients are allowed to send some limited messages to the server
    without prompting, these are called Well Known flows. By default
    these can be used to send log messages, or errors. A malicious
    client using these could fill up logs and disk space.

  - If you have custom Well Known Flows that perform interesting
    actions. You need to be aware that untrusted clients can call them.
    Most often this could result in a DoS condition, e.g. through a
    client sending multiple install failure or client crash messages.

In many environments this risk is unwarranted, so we suggest
implementing further authorization in the Enrollment Flow using some
information that only your client knows, to authenticate it before
allowing it to become a registered client.

Note that this does not give someone the ability to overwrite data from
another client, as client name collisions are protected.

## Server Security.

The http server is designed to be exposed to the Internet, but there is
no reason for the other components in the GRR system to be.

The Administration UI by default listens on all interfaces, and is
protected by only basic authentication configured via the --htpasswd
parameter. We strongly recommend putting the UI on SSL and IP limiting
the clients that can connect. The best way to do this normally is by
hosting it inside Apache via wsgi, using Apache to provide the SSL and
other protection measures.

## Auditing.

By default GRR currently only offers limited audit logs in the /var/log/
directory. However, the system is designed to allow for deployment of
extensive auditing capabilities through the API router.

Setting API.DefaultRouter configuration router to an API router that
will be doing necessary access checks allows for sensible access
control. I.e. another user must authorize access before someone is given
access to a machine, or an admin must authorize before a hunt is run.

In order to enable full access control, add

    AdminUI Context:
      API.DefaultRouter: ApiCallRouterWithApprovalChecks

to your configuration. Note that GRR will try to send approval emails so
you also need to set up email domain / SMTP server / …​

## Enabling SSL on the AdminUI.

The AdminUI supports SSL if it is configured. We don’t currently
generate certs to enable this by default as certificate management is
messy, but you can enable by adding to your config something like:

    AdminUI.enable_ssl: True
    AdminUI.ssl_cert_file: "/etc/ssl/certs/grr.crt"
    AdminUI.ssl_key_file: "/etc/ssl/private/grr.plain.key"

Note that SSL performance using this method may be average. If you have
a lot of users and a single AdminUI, you may get better performance
putting GRR behind an SSL reverse proxy such as Apache and letting it
handle the SSL.

## GRR Security Checklist.

  - Generate new CA/server keys on initial install. Back up these keys
    somewhere securely.

  - Ensure the GRR Administrative UI interface is not exposed to the
    Internet and is protected.

<!-- end list -->

  - Introduce controls on enrollment to protect the server from
    unauthorized clients.

  - Produce obfuscated clients.

  - Regenerate code and driver signing keys with passphrases.

  - Run the http server serving clients on a separate machine to the
    workers.

  - Introduce a stronger AdminUI sign in mechanism and use the
    FullAccessControlManager.

  - Ensure the Administrative UI is SSL protected

  - Ensure the database server is using strong passwords and is well
    protected.

## Managing the Datastore

GRR currently ships with a sharded SQLite database that is used by
default, and a MySQL Advanced datastore that may be a better choice if
you have significant in-house MySQL experience and resources.

# Performance

GRR is designed to scale linearly, but performance depends significantly
on the datastore implementation, how it is being run, and the hardware
it is running on.

## Large Scale Deployment

The [GRR server components](implementation.md#grr-component-overview)
should be distributed across multiple machines in any deployment where
you expect to have more than a few hundred clients, or even smaller
deployments if you plan on doing intensive hunting. The performance
needs of the various components are discussed
[below](component-performance-needs), and some real-world example
deployment configurations are [described in the
FAQ](faq.md#what-hardware-do-i-need-to-run-grr).

You should install the GRR package on all machines and use configuration
management (chef, puppet etc.) to:

  - Distribute the same grr-server.yaml to each machine

  - Control how many of each component to run on each machine (see next
    section for details)

## Running Multiple Copies of Components

For 3.1.0 and later, use a [systemd drop-in
override](http://askubuntu.com/questions/659267/how-do-i-override-or-configure-systemd-services)
to control how many copies of each component you run on each machine.
This can initially be done using:

    sudo systemctl edit grr-server

which creates "/etc/systemd/system/grr-server.service.d/override.conf".
You’ll want to turn this into a template file and control via puppet or
similar. An example override that just runs 3 workers looks like:

    [Service]
    ExecReload=
    ExecReload=/bin/systemctl --no-block reload grr-server@worker.service grr-server@worker2.service grr-server@worker3.service
    ExecStart=
    ExecStart=/bin/systemctl --no-block start grr-server@worker.service grr-server@worker2.service grr-server@worker3.service
    ExecStop=
    ExecStop=/bin/systemctl --no-block stop grr-server@worker.service grr-server@worker2.service grr-server@worker3.service

When starting multiple copies of the UI and the web frontend you also
need to tell GRR which ports it should be using. So if you want 10 http
frontends on a machine you would configure your systemd drop-in to start
10 copies and then set Frontend.port\_max so that you have a range of 10
ports from Frontend.bind\_port. (I.E. set Frontend.bind\_port to 8080
and Frontend.port\_max to 8089) You can then configure your load
balancer to distribute across that port range. AdminUI.port\_max works
the same way for the UI.

Prior to 3.1.0 the approach was to use config management tools to:

  - Manipulate the /etc/default/grr-\* files to enable the relevant
    services you want to run on each machine

  - Create new init scripts for components that should have multiple
    instances on each machine. e.g. If you want to run 20 workers you’d
    set up a puppet template to create 19 extra
    /etc/init/grr-worker\[1-19\].conf files. This will get easier when
    we have a docker cloud deployment, which is naturally suited to
    standing up many copies of services.

## Component Performance Needs

  - **Worker**: you will probably want to run more than one worker. In a
    large deployment where you are running numerous hunts it makes sense
    to run 20+ workers. As long as the datastore scales, the more
    workers you have the faster things get done. We previously had a
    config setting that forked worker processes off, but this turned out
    to play badly with datastore connection pools, the stats store, and
    monitoring ports so it was removed.

  - **HTTP frontend**: The frontend http server can be a significant
    bottleneck. By default we ship with a simple http server, but this
    is single process, written in python which means it may have thread
    lock issues. To get better performance you will need to run the http
    server with the wsgi\_server in the tools directory from inside a
    faster web server such as Apache. See section below for how. As well
    as having a better performing http server, if you are moving a lot
    of traffic you probably want to run multiple http servers. Again,
    assuming your datastore handles it, these should scale linearly.

  - **Web UI**: The admin UI component is usually under light load, but
    you can run as many as you want for redundancy. The more concurrent
    GRR users you have, the more instances you need. This is also the
    API server, so if you intend to use the API heavily run more.

## Performance-Critical Client Config Settings

  - **Foreman check frequency**: By default the
    foreman\_check\_frequency in the client configuration is set to 1
    hr. This variable controls how often a client checks if there are
    hunts scheduled for it. Increasing this number slows down how fast a
    hunt ramps up, which normalizes the load at the cost of making the
    hunt slower (this is useful in large deployments). Decreasing this
    number means clients pick up hunts sooner, but each foreman check
    incurs a penalty on the frontend server, as it must queue up a check
    against the rules.

## Datastore Performance

If you are not CPU bound on the individual components (workers, http
server) then the key performance differentiator will be the datastore.
Significant performance improvement work has been done on the MySQL
Advanced and SQLite datastores but more can still be done. Improvements
here will yield large gains, pull requests welcome :)

## Running the GRR HTTP Server In Apache

TBD. User contributions welcome. Using the wsgi hasn’t been thoroughly
tested. If you test, please send feedback to the dev list and we can try
and fix things.

## Running the GRR HTTP Server Behind Apache Reverse Proxy

Running apache as a reverse proxy in front of the GRR admin UI is a good
way to provide SSL protection for the UI traffic and also integrate with
corporate single sign on (if available), for authentication.

Buy an SSL certificate, or generate a self-signed one if you’re only
testing.

Place the public key into “/etc/ssl/certs/“ and ensure it’s world
readable

    chmod 644 /etc/ssl/certs/grr_ssl_certificate_filename.crt

Place the private key into “/etc/ssl/private” and ensure it is **NOT**
world readable

    chmod 400 /etc/ssl/private/grr_ssl_certificate_filename.key

Install apache2 and required modules

    apt-get install apache2
    a2enmod proxy
    a2enmod ssl
    a2enmod proxy_http

Disable any default apache files currently enabled (probably
000-default.conf, but check for others that may interfere with GRR)

    a2dissite 000-default

Redirect port 80 HTTP to 443 HTTPS

Create the file "/etc/apache2/sites-available/redirect.conf" and copy
the text below into it.

    <VirtualHost *:80>
        Redirect "/" "https://<your grr adminUI url here>"
    </VirtualHost>

Reverse Proxy GRR AdminUI Traffic

Create the file "/etc/apache2/sites-available/grr\_reverse\_proxy.conf"
and copy the text below into it.

    <VirtualHost *:443>
    SSLEngine On
    SSLCertificateFile /etc/ssl/certs/grr_ssl_certificate_filename.crt
    SSLCertificateKeyFile /etc/ssl/private/grr_ssl_certificate_filename.key
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
    </VirtualHost>

Enable the new apache files

    a2ensite redirect.conf
    a2ensite grr_reverse_proxy.conf

Restart apache

    service apache2 restart

  - NOTE: This reverse proxy will only proxy the AdminUI. It will have
    no impact on the agent communications on port 8080. It is advised to
    restrict access to the AdminUI at the network level.

# Scheduling Flows with Cron

The cron allows for scheduling flows to run regularly on the GRR server.
This is currently used to collect statistics and do cleanup on the
database. The cron runs as part of the workers.

# Customizing the client

The client can be customized for deployment. There are two keys ways of
doing this:

1.  Repack the released client with a new configuration.

2.  Rebuild the client from scratch (advanced users, set aside a few
    days the first time)

Doing a rebuild allows full reconfiguration, changing names and
everything else. A repack on the other hand limits what you can change.
Each approach is described below.

## Repacking the Client with a New Configuration.

Changing basic configuration parameters can be done by editing the
server config file (/etc/grr/server.local.yaml) to override default
values, and then using the config\_updater to repack the binaries. This
allows for changing basic configuration parameters such as the URL the
client reports back to.

Once the config has been edited, you can repack all clients with the new
config and upload them to the datastore using `grr_config_updater
repack_clients`

``` shell
db@host:$ sudo grr_config_updater repack_clients
Using configuration <ConfigFileParser filename="/etc/grr/grr-server.conf">

## Repacking GRR windows amd64 2.5.0.4 client
Packed to /usr/share/grr/executables/windows/installers/GRR_2.5.0.4_amd64.exe

## Uploading
Uploading Windows amd64 binary from /usr/share/grr/executables/windows/installers/GRR_2.5.0.4_amd64.exe
Uploaded to aff4:/config/executables/windows/installers/GRR_2.5.0.4_amd64.exe
db@host:$
```

Repacking works by taking the template zip file which are by default
installed to `/usr/share/grr/executables`, injecting relevant
configuration files, and renaming files inside the zip to match
requested names. This template is then turned into something that can be
deployed on the system by using the debian package builder on linux,
creating a self extracting zip on Windows, or creating a DMG on OSX.

After running the repack you should have binaries available in the UI
under manage binaries → installers and also on the filesystem under:

    /usr/share/grr/executables/windows/installers/
    /usr/share/grr/executables/osx/installers/
    /usr/share/grr/executables/linux/installers/

## Repacking Clients: Signing installer binaries

You can also use the grr\_client\_build tool to repack individual
templates and control more aspects of the repacking, such as signing.
For signing to work you need to follow these instructions:

  - [Setting up for RPM
    signing](linuxclient.md#setting-up-for-linux-rpm-signing)

  - [Setting up for Windows EXE
    signing](windowsclient.md#setting-up-for-windows-exe-signing)

Then add the --sign parameter to the repack
    command:

    grr_client_build repack --template path/to/grr-response-templates/templates/grr_3.1.0.2_amd64.xar.zip --output_dir=/tmp/test --sign

To repack and sign multiple templates at once, see the next section.

## Repacking Clients With Custom Labels: Multi-Organization Deployments

Each client can have a label "baked in" at build time that allows it to
be identified and hunted separately. This is especially useful when you
want to deploy across a large number of separate organisations. You
achieve this by creating a config file that contains the unique
configuration you want applied to the template. A minimal config that
just applies a label would contain:

    Client.labels: [mylabel]

and be written into repack\_configs/mylabel.yaml.

Then you can call repack\_multiple to repack all templates (or whichever
templates you choose) with this configuration and any others in the
repack\_configs directory. An installer will be built for each
    config:

    grr_client_build repack_multiple --templates /path/to/grr-response-templates/templates/*.zip --repack_configs /path/to/repack_configs/*.yaml --output_dir=/grr_installers

To sign the installers (RPM and EXE), add --sign.

## Building the Client.

There’s no need to rebuild the client for testing or small deployments,
use [repacking](#repacking-the-client-with-a-new-configuration) instead.

OS X and Linux clients are built on every commit using travis (after
tests pass), and the results are stored on Google cloud storage. Windows
templates are [built on appveyor and the results are downloadable via
their website](https://ci.appveyor.com/project/destijl/grr). You can use
these templates directly, but be aware you are trusting the travis and
appveyor infrastructure to be secure.

If you want to build your own templates because you have customised
code, use the [vagrant](https://www.vagrantup.com/) build system. You’ll
need a copy of the GRR source:

    git clone https://github.com/google/grr.git

and the latest versions of [Vagrant](https://www.vagrantup.com/) and
VirtualBox installed. If you reboot the provided linux VM’s and get the
new kernel you’ll need to update the VirtualBox guest additions. You can
use [vagrant-vbguest](https://github.com/dotless-de/vagrant-vbguest) to
do this automatically, but you should download and verify the hash on
the guest additions yourself (vagrant-vbguest downloads over HTTP and
doesn’t verify hash).

OS X and windows require some extra work, see here for instructions:

  - [Building the OS X client](osxclient.md)

  - [Building the Windows client](windowsclient.md)

Once you have your vagrant VMs setup (only necessary for OS X, linux
will download VMs automatically), you can build templates for all linux
OSes just by running make. If you’re building for OS X as well, you’ll
run this once on linux and once on apple hardware. Windows requires more
work as described
[here](https://github.com/google/grr-doc/blob/markdown/windowsclient.md#building-templates).

    cd vagrant
    make templates

To get clean VMs and re-run the provisioning for all linux and OS X VMs
you can use:

    make vmclean

Once you have templates built you need to follow the repacking
instructions above to turn them into installers.

## Client Configuration.

Configuration of the client is done during the packing/repacking of the
client. The process looks like:

1.  For the client we are packing, find the correct context and
    platform, e.g. `Platform: Windows` `Client Context`

2.  Extract the relevant configuration parameters for that context from
    the server configuration file, and write them to a client specific
    configuration file e.g. `GRR.exe.yaml`

3.  Pack that configuration file into the binary to be deployed.

When the client runs, it determines the configuration in the following
manner based on --config and --secondary\_configs arguments that are
given to it:

1.  Read the config file packed with the installer, default:
    `c:\windows\system32\GRR\GRR.exe.yaml`

2.  GRR.exe.yaml reads the Config.writeback value, default:
    `reg://HKEY_LOCAL_MACHINE/Software/GRR` by default

3.  Read in the values at that registry key and override any values from
    the yaml file with those values.

Most parameters are able to be modified by changing parameters and then
restarting GRR. However, some configuration options, such as
`Client.name` affect the name of the actual binary itself and therefore
can only be changed with a repack on the server.

Updating a configuration variable in the client can be done in multiple
ways:

1.  Change the configuration on the server, repack the clients and
    redeploy/update them.

2.  Edit the yaml configuration file on the machine running the client
    and restart the client.

3.  Update where Config.writeback points to with new values, e.g. by
    editing the registry key.

4.  Issue an UpdateConfig flow from the server (not visible in the UI),
    to achieve 3.

In practice, you should nearly always do 3 or 4.

As an example, to reduce how often the client polls the server to every
300 seconds, you can update the registry as per below, and then restart
the service:

``` shell
C:\Windows\System32\>reg add HKLM\Software\GRR /v Client.poll_max /d 300

The operation completed successfully.
C:\Windows\System32\>
```

## Common Client Configuration Options.

The client has numerous configuration parameters that control its
behavior, the following explains some key ones you might want to change:

<table>
<colgroup>
<col width="15%" />
<col width="85%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>Client Behavior Keys</p></td>
<td><p>Keys which affect behavior of the client. Should take affect on client restart.</p>
<dl>
<dt>Client.poll_max</dt>
<dd><p>Maximum number of seconds between polls to the server.</p>
</dd>
<dt>Client.foreman_check_frequency</dt>
<dd><p>How often to check for foreman jobs (hunts).</p>
</dd>
<dt>Client.rss_max</dt>
<dd><p>Maximum memory for the client to use.</p>
</dd>
<dt>Client.control_urls</dt>
<dd><p>The list of URLs to contact the server on.</p>
</dd>
<dt>Client.proxy_servers</dt>
<dd><p>A list of proxy servers to try.</p>
</dd>
<dt>Logging.verbose</dt>
<dd><p>Enable more verbose logging.</p>
</dd>
<dt>Logging.engines</dt>
<dd><p>Enable or disable syslog, event logs or file logs.</p>
</dd>
<dt>Logging.path</dt>
<dd><p>Where log files get written to.</p>
</dd>
</dl></td>
</tr>
</tbody>
</table>

<table>
<colgroup>
<col width="15%" />
<col width="85%" />
</colgroup>
<tbody>
<tr class="odd">
<td><p>Obfuscation Related Keys</p></td>
<td><p>Keys you might want to change to affect obfuscation, these will require a rebuild.</p>
<dl>
<dt>Client.name</dt>
<dd><p>The base name of the client. Changing this to Foo will change the running binary to Foo.exe and Fooservice.exe on Windows.</p>
</dd>
<dt>Client.config_key</dt>
<dd><p>The registry key to store config data on Windows</p>
</dd>
<dt>Client.control_urls</dt>
<dd><p>The list of URLs to contact the server on.</p>
</dd>
<dt>Client.plist_path</dt>
<dd><p>Where to store the plist on OSX.</p>
</dd>
<dt>MemoryDriver.display_name</dt>
<dd><p>Description of the service used for the memory driver on Windows</p>
</dd>
<dt>MemoryDriver.service_name</dt>
<dd><p>Name of the service used for the memory driver on Windows</p>
</dd>
<dt>MemoryDriver.install_write_path</dt>
<dd><p>Path to write the memory driver to.</p>
</dd>
<dt>Nanny.service_name</dt>
<dd><p>Name of the Windows service the nanny runs as.</p>
</dd>
<dt>Nanny.service_description</dt>
<dd><p>Description of the Windows service the nanny runs as.</p>
</dd>
<dt>ClientBuilder.console</dt>
<dd><p>Affects whether the installer is silent.</p>
</dd>
</dl></td>
</tr>
</tbody>
</table>

For a full list of available options you can run `grr_server
--config-help` and look for `Client`, `Nanny` and `Logging` options.

# Deploying The Client

For first-time deployment, GRR Clients need to be installed using
existing package management systems for each platform. For Windows the
installer is a self-extracting executable which can be deployed using
standard tools, such as SCCM, but some smaller networks use approaches
that require an MSI. In this case we suggest using one of the various
third-party tools for creating .msi’s from .exe’s, detailed instructions
can be found
[here](http://grr-response.blogspot.com/2014/12/wrapping-grr-installers-as-msi-file.html).

# Deploying Custom Drivers and Code.

Drivers, binaries or python code can be pushed from the server to the
clients to enable new functionality. This has a number of use cases,
such as:

  - Upgrades. When you want to update the client you need to be able to
    push new code.

  - Drivers. If you want to load a driver on the client system to do
    memory analysis, you may need a custom driver per system (e.g. in
    the case of Linux kernel differences.)

  - Protected functionality. If you have code that you want to deploy to
    deal with a specific case, you may not want that to be part of the
    client, and should only be deployed to specific clients.

The code that is pushed from the server must be signed by the
corresponding private key for `Client.executable_signing_public_key` for
python and binaries or the corresponding private key for
Client.driver\_signing\_public\_key for drivers. These signatures will
be checked by the client to ensure they match before the code is used.

What is actually sent to the client is the code or binary wrapped in a
protobuf which will contain a hash, a signature and some other
configuration data.

To sign code requires use of config\_updater utility. In a secure
environment the signing may occur on a different box from the server,
but the examples below show the basic example.

## Deploying Arbitrary Python Code.

To execute an arbitrary python blob, you need to create a file with
python code that has the following attributes:

  - Code in the file must work when executed by exec() in the context of
    a running GRR client.

  - Any return data that you want sent back to the server should be
    stored encoded as a string in a variable called
    "magic\_return\_str".

E.g. as a simple example. The following code modifies the clients
poll\_max setting and pings test.com.

``` python
import commands
status, output = commands.getstatusoutput("ping -c 3 test.com")
config_lib.CONFIG.Set("Client.poll_max", 100)
config_lib.CONFIG.Write()
magic_return_str = "poll_max successfully set. ping output %s" % output
```

This file then needs to be signed and converted into the protobuf format
required, and then needs to be uploaded to the data store. You can do
this using the following command line.

``` shell
grr_config_updater upload_python --file=myfile.py --platform=windows
```

The uploaded files live by convention in aff4:/config/python\_hacks and
are viewable in the Manage Binaries section of the Admin UI.

The ExecutePythonHack Flow is provided for executing the file on a
client. This is visible in the Admin UI under Administrative flows if
Advanced Mode is enabled.

> **Note**
> 
> Specifying arguments to a PythonHack is possible as well through the
> py\_args argument, this can be useful for making the hack more
> generic.

## Deploying Drivers

Drivers are currently used in memory analysis. By default we use drivers
developed and released by the Rekall team named "pmem". We currently
have Apache Licensed, tested drivers for OSX, Linux and Windows. GRR
Does not currently support loading drivers which are not designed to
work with Rekall.

The drivers are distributed with GRR but are also available from the
Rekall project site in binary form at <http://www.rekall-forensic.com/>.
(To extract just the drivers you can just unzip them from the winpmem
binary itself).

Deploying a driver works much the same as deploying python code. We sign
the file, encode it in a protobuf and upload it to a specific place in
the GRR datastore. There is a shortcut to upload the memory drivers
shipped with GRR using config updater. This will place the drivers
shipped with GRR from their default locations into the expected
location.

``` shell
db@host: ~$ sudo grr_config_updater load_memory_drivers
Using configuration <ConfigFileParser filename="/etc/grr/grr-server.conf">
uploaded aff4:/config/drivers/darwin/memory/osxpmem
uploaded aff4:/config/drivers/windows/memory/winpmem.32.sys
uploaded aff4:/config/drivers/windows/memory/winpmem.64.sys
db@host:$
```

If this worked you should now see them under Manage Binaries in the
Admin UI.

If you need to add a new driver or add a custom install you can use the
upload memory driver
functionality:

``` shell
db@host:~$ sudo grr_config_updater upload_memory_driver --file=/path/to/my_special_pmem.kext.tgz --platform=windows --arch amd64 --dest aff4:/config/drivers/osx/memory/pmem
```

If you need to customize some property of the driver you can easily
inject configuration parameters into the above command line (this *must*
be done before the `upload_memory_driver` command). For example, if you
recompiled the driver to present a different device name on the
client:

``` shell
db@host:~$ sudo grr_config_updater -pMemoryDriver.device_path=/dev/my_pmem_device upload_memory_driver --file=/path/to/my_special_pmem.kext.tgz --platform=windows  --arch amd64 --dest aff4:/config/drivers/osx/memory/pmem
```

> **Note**
> 
> The signing we discuss here is independent of Authenticode driver
> signing, which is also required by modern 64 bit Windows
> distributions.

Deploying this driver would normally be done using the LoadMemoryDriver
flow.

## Building a Linux memory driver

Determine the kernel version of the system the GRR client is running on
e.g.

    3.13.0-49-generic

Find a reprentative build machine e.g. Ubuntu and install the
corresponding kernel headesr:

    sudo apt-get install linux-headers-3.13.0-49-generic

Clone rekall and traverse into the linux driver source directory:

    git clone https://github.com/google/rekall.git
    cd rekall/tools/linux

edit Makefile and set KVER to the kernel version

    KVER ?= "3.13.0-49-generic

Build the pmem driver:

    make pmem

Deploy pmem.ko as a driver.

## Deploying Executables.

The GRR Agent provides an ExecuteBinaryCommand Client Action which
allows us to send a binary and set of command line arguments to be
executed. The binary must be signed using the executable signing key
(config option PrivateKeys.executable\_signing\_private\_key).

To sign an exe for execution use the config updater
script.

``` shell
db@host:$ grr_config_updater upload_exe --file=/tmp/bazinga.exe --platform=windows
Using configuration <ConfigFileParser filename="/etc/grr/grr-server.conf">
Uploaded successfully to /config/executables/windows/installers/bazinga.exe
db@host:$
```

This uploads to the installers directory by default. But you can
override with the --dest\_path option.

This file can then be executed with the LaunchBinary flow which is in
the Administrative flows if Advanced Mode is enabled.

# Client Robustness Mechanisms

We have a number of mechanisms built into the client to try and ensure
it has sensible resource requirements, doesn’t get out of control, and
doesn’t accidentally die. We document them here.

## Heart beat

The client process regularly writes to a registry key (file on Linux and
OSX) with a timer. The nanny process watches this registry key called
HeartBeat, if it notices that the the client hasn’t updated the
heartbeat in the time allocated by UNRESPONSIVE\_KILL\_PERIOD (default 3
minutes), the nanny will assume the client has hung and will kill it. In
Windows we then rely on the Nanny to revive it, on Linux and OSX we rely
on the service handling mechanism to do so.

## Transaction log

When the client is about to start an action it writes to a registry key
containing information about what it is about to do. If the client dies
while performing the action, when the client gets restarted it will send
an error along with the data from the transaction log to help diagnose
the issue.

One tricky thing with the transaction log is the case of Bluescreens or
kernel panics. Writing to the transaction log will write a registry key
on Windows, but registry keys are not flushed to disk immediately.
Therefore, writing a transaction log, and then getting a hard BlueScreen
or kernel panic, the transaction log won’t be persistent, and therefore
the error won’t be sent. We work around this by adding a Flush to the
transaction log when we are about to do dangerous transactions, such as
loading a memory driver. But if the client dies during a transaction we
didn’t deem as dangerous, it is possible that you will not get a crash
report.

## Memory limit

We have a hard and a soft memory limit built into the client to stop it
getting out of control. The hard limit is enforced by the nanny, if the
client goes over that limit it will be hard killed. The soft limit is
enforced by the client, if the limit is exceeded the client will stop
retrieving new work to do. Once it has finished its current work it will
die cleanly.

Default soft limit is 500MB, but GRR should only use about 30MB. Some
volatility plugins can use a lot of memory so we try to be generous.
Hard limit is double the soft limit. This is configurable from the
config file.

## CPU limit

A ClientAction can be transmitted from the server with a specified CPU
limit, this is how many seconds the action can use. If the action uses
more than that it will be killed. The actual implementation is a little
more complicated. An action can run for 3 minutes using any CPU it wants
before being killed by nanny. However actions that are good citizens
(normally the dangerous ones) will call the Progress() function
regularly. This function checks if limit has been exceeded and will
exit.

# Crashes

The client shouldn’t ever crash…​ but it does because making software is
hard. There are a few ways in which this can happen, all of which we try
and catch, record and make visible to allow for debugging. In the UI
they are visible in two ways, in "Crashes" when a client is selected,
and in "All Client Crashes". These have the same information but the
client view only shows crashes for the specific client.

Each crash should contain the reason for the crash, optionally it may
contain the flow or action that caused the crash. In some cases this
information is not available because the client may have crashed when it
wasn’t doing anything or in a way where we could not tie it to the
action. See [Client Robustness
Mechanisms](#_client_robustness_mechanisms) for a discussion of this.

This data is also emailed to the email address configured in the config
as Monitoring.alert\_email

## Crash Types

### Crashed while executing an action

Often seen with an error "Client killed during transaction". This means
that while handling a specific action, the client died, the nanny knows
this because the client recorded the action it was about to take in the
Transaction Log before starting it. When the client restarts it picks up
this log and notifies the server of the crash.

Causes

  - Client segfaults, could happen in native code such Sleuth Kit or
    psutil.

  - Hard reboot while the machine was running an action where the client
    service didn’t have a chance to exit cleanly.

### Unexpected child process exit\!

This means the client exited, but the nanny didn’t kill it.

Causes

  - Uncaught exception in python, very unlikely due to the fact that we
    catch Exception for all client actions.

### Memory limit exceeded, exiting

This means the client exited due to exceeding the soft memory limit.

Causes

  - Client hits the soft memory limit. Soft memory limit is when the
    client knows it is using too much memory but will continue operation
    until it finishes what it is doing.

### Nanny Message - No heartbeat received

This means that the Nanny killed the client because it didn’t receive a
Heartbeat within the allocated time.

Causes

  - The client has hung, e.g. locked accessing network file

  - The client is performing an action that is taking longer than it
    should.

# Notifiying Users of Updates or Issues via the GUI

GRR has the ability to display a notification similar to the yellow
[Chrome Infobar](http://www.chromium.org/user-experience/infobars). This
can be useful if you need to let users know about new functionality,
updates, problems, downtime etc. For now it requires console access to
set a new
    notification.

    flow.GRRFlow.StartFlow(flow_name="SetGlobalNotification", type="WARNING", content="NOTE: This is a one-time warning. To hide this message click on X in the right corner of this panel.", link="http://company.com/moreinfo", token=rdfvalue.ACLToken(username="myuser"))

To remove all notifications:

    aff4.FACTORY.Delete("aff4:/config/global_notifications")

d

1.  Read only access many not give direct code exec, but may well
    provide it indirectly via read access to important keys and
    passwords on disk or in memory.
