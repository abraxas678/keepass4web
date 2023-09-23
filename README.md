![Build](https://travis-ci.org/lixmal/keepass4web.svg?branch=master)
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [KeePass4Web](#keepass4web)
  - [FEATURES](#features)
  - [INSTALL](#install)
  - [BUILD FRONTEND](#build-frontend)
  - [MODULE INSTALLATION](#module-installation)
  - [CONFIGURATION](#configuration)
  - [DEPLOYMENT](#deployment)
    - [Container](#container)
    - [Classic](#classic)
        - [Running apache2 using mod_perl2/Plack with TLS:](#running-apache2-using-mod_perl2plack-with-tls)
        - [Using the standalone server](#using-the-standalone-server)
        - [Open `https://<domain>/keepass/` (notice the trailing slash)](#open-httpsdomainkeepass-notice-the-trailing-slash)
        - [Refer to Dancer2::Manual::Deployment for more options.](#refer-to-dancer2manualdeployment-for-more-options)
  - [BUNDLING](#bundling)
  - [BACKENDS](#backends)
    - [Authentication](#authentication)
        - [LDAP](#ldap)
        - [Htpasswd](#htpasswd)
        - [PAM (planned)](#pam-planned)
        - [SQL (planned)](#sql-planned)
    - [Database](#database)
        - [Filesystem](#filesystem)
        - [Seafile](#seafile)
        - [LWP](#lwp)
        - [Dropbox](#dropbox)
        - [WebDAV (planned)](#webdav-planned)
  - [MISCELLANEOUS](#miscellaneous)
  - [LIMITATIONS](#limitations)
  - [BUGS / CAVEATS / TODO](#bugs--caveats--todo)
  - [APP DETAILS / BACKGROUND](#app-details--background)
    - [Sequence of client/server operations](#sequence-of-clientserver-operations)
    - [Packages used](#packages-used)
        - [Libraries / Packages](#libraries--packages)
        - [Perl modules](#perl-modules)
          - [Core](#core)
          - [Backend LDAP](#backend-ldap)
          - [Backend Htpasswd](#backend-htpasswd)
          - [Backend Seafile](#backend-seafile)
          - [Backend LWP](#backend-lwp)
          - [Backend Dropbox](#backend-dropbox)
          - [Bundled modules, may become external](#bundled-modules-may-become-external)
  - [COPYRIGHT AND LICENSING](#copyright-and-licensing)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# KeePass4Web

A mobile-friendly web application which serves KeePass database entries on a web frontend.

Written in Perl and JavaScript.


## FEATURES

- Users need to authenticate with one of the auth backends (LDAP, Htpasswd, SQL, ...) before they can open a database
- Server can fetch databases from various locations (Filesystem, Seafile, Dropbox, ...)
- Either all users get access to the same database or each user gets access to his/her own
- Doesn't save master password, uses a new and unique encryption key to cache the database
- Caches encrypted databases in shared memory (so it works with multiple web server workers)
- Encryption key is stored in the kernel keyring and therefore doesn't swap to disk
- Passwords, protected fields and files are encrypted separately (also the ones in history). The web server only decrypts requested information. This way other passwords don't stay in memory in plain text and don't leave the server
- Server revokes encryption keys after a configurable user idle time, effectively removing access to the cached database
- Web interface offers entry search and access to files stored inside the database. Also displays custom entry icons
- Highly configurable


![Login](doc/img/login.png)

![App](doc/img/app.png)


## INSTALL

- From container image:
See [DEPLOYMENT](#deployment)

- From source:
    - Clone the repo to some dir
        > git clone https://github.com/lixmal/keepass4web.git

        > cd keepass4web

    - Follow [BUILD FRONTEND](#build-frontend), [MODULE INSTALLATION](#module-installation), [CONFIGURATION](#configuration), [DEPLOYMENT](#deployment) in that order


## BUILD FRONTEND

The minified, bundled file will be written to public/scripts/bundle.js

- Install Node/npm, e.g. for Ubuntu
    > sudo apt-get install npm

- Install js modules
    > npm install

- Copy bootstrap font files
    > cp node_modules/bootstrap/fonts/* public/fonts/

- Build js bundle
    > npm run build

- For a non-uglified version you can run
    > npm run dev

## MODULE INSTALLATION

E.g. for Ubuntu 22.04 with mod_perl2:

- Install distro packages
    > sudo apt-get install build-essential libkeyutils-dev libkeyutils1 libmagic1 libmagic-dev libapache2-mod-perl2 cpanminus

- Install dependencies with all backends, the recommended modules (for performance) and the suggested session engine (`Cookie`)
    > cpanm --sudo --installdeps . --with-all-features --with-recommends --with-suggests

- Alternatively, install dependencies with selected backends only
    > cpanm --sudo --installdeps . --with-feature Dropbox --with-feature LDAP --with-recommends


## CONFIGURATION

- Copy or rename `config.yml` to `config_local.yml`
    > cp config.yml config_local.yml

- Make changes in `config_local.yml`. Settings in `config_local.yml` override those in `config.yml`

- Change `session_cookie_key` to a **long** and **random** value if using `Cookie` in `session`, e.g.
    > pwgen -ysN1 128


## DEPLOYMENT

### Container

See [GitHub Packages](https://ghcr.io/lixmal/keepass4web)

The image ships with the default config in `/conf/config.yml`, which should be overwritten with a mount/volume.

The app makes use of the [Linux kernel keyring](https://man7.org/linux/man-pages/man7/keyrings.7.html).

The keyring is currently not namespaced, hence container tooling deactivate the specific syscalls by default.
To make the app run you will need to activate the syscalls by creating a custom seccomp profile and passing the path to the container runtime:

- [Docker](https://docs.docker.com/engine/security/seccomp/)
- [podman](https://docs.podman.io/en/v4.6.0/markdown/options/seccomp-policy.html)

A base file for extension can be found [here](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json), see the `syscalls` section.

The required syscalls are:

- keyctl
- add_key
- request_key


**Make sure no other containers are running under the same user, or they will be able to access keys stored for keepass4web**.

This is best achieved by running rootless containers with a dedicated user for keepass4web.

- [Docker](https://docs.docker.com/engine/security/rootless/)
- [podman](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)


### Classic

Running this app on a web server with mod_perl2 or fcgi is **recommended** but running as standalone app is possible as well (with Dancer2's capabilities).

- Create the log directory (as defined in `config_local.yml`)
    > sudo mkdir /var/log/keepass4web/

- The directory the app lives in has to be readable by the user running the web server, e.g.
    > sudo chown -R root:www-data /opt/keepass4web/ /var/log/keepass4web/

    > chmod g+r -R /opt/keepass4web/

- Addtionally, it needs write permissions on the log directory, e.g.
    > chmod g+w /var/log/keepass4web/

- Remove permissions on sensitive data for everyone else
    > chmod o= /opt/keepass4web/config*.yml /var/log/keepass4web/

- For apache, enable the perl mod and ssl
    > sudo a2enmod perl

    > sudo a2enmod ssl

    > sudo a2ensite default-ssl


##### Running apache2 using mod_perl2/Plack with TLS:

Example config default-ssl:

```apache
PerlSwitches -I/opt/keepass4web/lib/
PerlModule KeePass4Web::Apache2
PerlPostConfigHandler KeePass4Web::Apache2::post_config

<IfModule mod_ssl.c>
    <VirtualHost _default_:443>
    ServerName example.org

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
    SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

    PerlOptions +Parent
    <Location /keepass/>
        SetHandler perl-script
        PerlResponseHandler Plack::Handler::Apache2
        PerlSetVar psgi_app /opt/keepass4web/bin/app.psgi
    </Location>


    </VirtualHost>
</IfModule>
```

##### Using the standalone server

- Run (as correct user)
    > plackup bin/app.psgi --port 8080 --host localhost

- Or
    > DANCER_PORT=8080 DANCER_SERVER=localhost bin/app.psgi

- Options for plackup can be found in `man plackup` or [online](https://metacpan.org/pod/distribution/Plack/script/plackup)

- As there is no TLS, it is recommonded to run a front-end web server with reverse proxy, example config for apache:
    ```apache
    <IfModule mod_ssl.c>
        <VirtualHost _default_:443>
        ServerName example.org

        SSLEngine on
        SSLCertificateFile    /etc/ssl/certs/ssl-cert-snakeoil.pem
        SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

        ProxyPass        /keepass/ http://localhost:8080/
        ProxyPassReverse /keepass/ http://localhost:8080/


        </VirtualHost>
    </IfModule>
    ```

##### Open `https://<domain>/keepass/` (notice the trailing slash)

##### Refer to [Dancer2::Manual::Deployment](https://metacpan.org/pod/Dancer2::Manual::Deployment) for more options.

## BUNDLING

Output will be a `KeePass4Web-{VERSION}.tar.gz` file, which includes all files required to run the app but without the development/build files

- Follow `BUILDING` first, then run the perl make file
    > perl Makefile.PL

- Bundle the app to a tar
    > make dist

- Clean up afterwards
    > make clean



## BACKENDS

### Authentication

Credentials may be further passed to the database backend via `auth_reuse_cred` config option.
This way users don't have to enter their credentials twice (for auth backend and database backend) if they are identical.
Useful when the database backend server uses the same auth backend internally.

##### LDAP

Attempts to authenticate the user against an (external, not built-in) LDAP server (Microsoft AD, 389 Directory Server, OpenLDAP, ...)

##### Htpasswd

Authentication using apache htpasswd files.
Support for plain, sha1, crypt, md5 and bcrypt. Only bcrypt is considered secure.

##### PAM (planned)

##### SQL (planned)

### Database

##### Filesystem

Grabs the KeePass database from the local filesystem. No support for local key files if configured statically (in `config_local.yml`).
Can get database and key file location from auth backend.
Web server needs read access to the files.

##### Seafile

KeePass database is stored in the private cloud.

Locations for the database and the key file can be configured in `config_local.yml` (global for all users) or fetched from auth backend (individually per user).
Location syntax is '\<repository-id>/\<path-to-database>', see `config_local.yml` for an example.

The server uses the (possibly auth backend) credentials to fetch a token from Seafile which is used for consequent requests.
No user credentials are saved anywhere at any time!

Right now the server does all Seafile requests. It is planned to migrate to a model where the client fetches the token from the Seafile server (if on the same domain) and passes it to the app server.
Logging into some service on behalf of the user is an anti-pattern, therefore it is not recommended for non private servers.

##### LWP

Backend to fetch database from http, ftp or any other protocol supported by the LWP module collection (see [LWP NETWORK SUPPORT](http://search.cpan.org/dist/libwww-perl/lib/LWP.pm#NETWORK_SUPPORT)).
Additional protocols can be added by installing corresponding modules (see [LWP::Protocol modules](https://metacpan.org/search?p=1&q=LWP%3A%3AProtocol%3A%3A&search_type=modules&size=30)).
Supports per-user database and key file location from auth backend. No support for key files if configured statically (in `config_local.yml`).
Basic auth is supported for http, but only globally (same for all users, even if urls differ). Otherwise it would be necessary to store the user credentials in the session, which might be not a good idea.
Database upload (saving) is only implemented for http right now.

Username and password may also be supplied in the form of `ftp://username:password@example.org/db.kdbx`

##### Dropbox

First you need to register the app with Dropbox: [Create App](https://www.dropbox.com/developers/apps/create).
Choose the type of access you need, give it a name (e.g. `KeePass4Web`).
Add an redirect url pointing to you application, followed by `callback`, e.g. `https://example.org/keepass/callback`.
Optionally, generate an access token.
Putting the token into the config will limit all users to that one Dropbox account (unless they open the Dropbox link by themselves, for which they would need to know the app key).
Put the displayed app key, the app secret and the redirect url into `config_local.yml`.

Now, once users log into the web application, they will be redirected to the Dropbox login page (unless already logged in).
After logging in and granting the app access to Dropbox, they will be redirected back to the app.

The backend also can fetch key files from Dropbox, if per-user databases are supported by the auth backend.

For the format to use in `config_local.yml`/`db_location` or the auth backend see the [Dropbox HTTP doc](https://www.dropbox.com/developers/documentation/http/documentation) under `Path formats` or [HTTP download](https://www.dropbox.com/developers/documentation/http/documentation#files-download) next to `Paramaters`

##### WebDAV (planned)


## MISCELLANEOUS

- Show currently used shared segments
    > sudo ipcs

- Removing segments (effectively closing user databases)
    > sudo ipcrm -M `key`

- Show kernel keyrings in use (as root)
    > sudo cat /proc/keys

    > sudo cat /proc/key-users

- Adding users to .htpasswd, using bcrypt (needs apache2-utils/httpd-tools)
    > touch .htpasswd               # create file

    > htpasswd -B .htpasswd <username>

    > sudo chown root:www-data .htpasswd # change group

    > chmod g+r,o-rwx .htpasswd     # remove permission of others, add read to webserver user


## LIMITATIONS

- Doesn't support [kdbx 4](https://keepass.info/help/kb/kdbx_4.html) yet
- KeePass databases are read-only for now
- Caching of KeePass databases happens in SysV IPC shared memory, whose maximum size depends on the OS. Defined by `shmall` and `shmmni` kernel variables
    > sudo cat /proc/sys/kernel/shmmni

    > sudo cat /proc/sys/kernel/shmall
- Limits of kernel keyring apply
- Right now all cached databases are seralised and deserialised together. This means more *simultaneously active* users will make fetching databases from IPC slower for every user. A better approach would be using one shared segment per user, which would make one roundtrip perpetual


## BUGS / CAVEATS / TODO

- Using mod_perl, apache may create two kernel session keyrings, because it restarts directly after startup, effectively executing KeePass4Web::KeePass twice

- Log may have 'Key has been revoked' messages: happens when session keyring gets revoked once user (who (re)started the server) logs out. Please file a bug report in this case.

- More tests


## APP DETAILS / BACKGROUND
### Sequence of client/server operations

```
Client                                                       Server


Load website /
                              request KeePass tree
                              -------------------->

                                                        Check sesssion

                              not authenticated
                              <--------------------

Redirect to /user_login
Show credentials dialog

                              user credentials
                              -------------------->

                                                        User auth (LDAP, SQL, ...)

                                           login OK
                              <--------------------

Redirect to /backend_login
Show backend login dialog

                              backend credentials
                              -------------------->

                                                        Init DB backend / receive backend token
                                           login OK
                              <--------------------

Redirect to /db_login
Show KeePass password dialog

                              KeePass credentials
                              -------------------->
                                                        Possibly decrypt backend repo
                                                        Get KeePass database from backend
                                                        Possibly get Key file from backend
                                                        Decrypt KeePass database with master key + key file
                                                        Encrypt all password fields
                                                        Encrypt serialised string with newly generated key
                                                        Put encryption key into kernel keyring
                                                        Write keyring ids to session
                                                        Put encrypted database into IPC shared memory
                                      decryption OK
                              <--------------------

Redirect to /
                              request KeePass tree
                              -------------------->

                                                        Get database from IPC shared memory
                                                        Get encryption key from session
                                                        Decrypt database with key

                                  Send KeePass tree
                              <--------------------
Show KeePass tree

...

Password request by user
                              Request pw entry
                              -------------------->

                                                        Get keyring id from session
                                                        Get encryption key from kernel keyring
                                                        Get database from IPC shared memory
                                                        Decrypt database
                                                        Decrypt requested password

                                  Send pw entry
                              <--------------------
Show cleartext pw



...


Page reload

                              Request KeePass tree
                              -------------------->
                                                        Get database from IPC shared memory

                                  Send KeePass tree
                              <--------------------
Show KeePass tree

```

### Packages used

##### Libraries / Packages

- build-essential *(building XS modules)*
- libkeyutils-dev
- libkeyutils1
- libapache2-mod-perl2 *(if running mod_perl2 with apache2)*
- libmagic1
- libmagic-dev
- cpanminus *(module installation)*

##### Perl modules

###### Core
- Kernel::Keyring
- Dancer2
- Dancer2::Plugin::Ajax
- Dancer2::Session::Cookie *(default session engine, `Cookie` in config)*
- IPC::ShareLite
- File::KeePass
- Crypt::URandom
- File::LibMagic
- Sereal::Encoder
- Sereal::Decoder
- Crypt::Mode::CBC
- Crypt::Mac::HMAC
- URI::Escape

###### Backend LDAP
- Net::LDAP

###### Backend Htpasswd
- Crypt::Eksblowfish::Bcrypt *(bcrypt)*
- Authen::Htpasswd  *(md5, sha1, crypt, plain)*

###### Backend Seafile
- JSON
- REST::Client
- URI::Escape

###### Backend LWP
- LWP::UserAgent
- HTTP::Request::Common
- URI::Escape

###### Backend Dropbox
- WebService::Dropbox

###### Bundled modules, may become external
- File::KeePass::Web
- Auth::LDAP
- Seafile::Client::REST



## COPYRIGHT AND LICENSING

This software is copyright (c) by Viktor Liu.
It is released under the terms of the GPL version 3.

Most of the icons in the `public/img/icons` directory are released under the LGPL version 2, the licence can be found in the same directory.
The remaining icons are public domain.
As these icons are the same as the ones used by the original KeePass software, you can refer to the info there: [Icon Acknowledgements](http://keepass.info/help/base/credits.html#icons).

The Seafile logo is copyright (c) by Seafile Ltd.
