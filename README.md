# dinit-userservd

v0.91.0 (pre-alpha release)

This is a daemon and a PAM module to handle user services management with the
`dinit` init system and service manager (https://github.com/davmac314/dinit).

It was created for the needs of the Chimera Linux project. Environments that
are significantly different from Chimera's may experience problems and are not
officially supported; feature requests related to such environments will not
be addressed.

Community patches addressing such features are welcome, provided they are not
disruptive and/or introduce excessive complexity.

## Purpose

As the name implies, the purpose of the project is to provide convenient
handling of user services. There are many things one might want to manage
through user services. This includes for instance the D-Bus session bus
or a sound server.

Thanks to the project, one can have user services that are automatically
spawned upon first login and shut down upon last logout. It also takes
care of some extra adjacent functionality that is handy to have.

## Setup

Build and install the project. It uses [Meson](https://mesonbuild.com/) and
follows the standard Meson workflow. Example:

```
$ mkdir build && cd build
$ meson .. --prefix=/usr
$ ninja all
$ sudo ninja install
```

The dependencies are:

1) A POSIX-compliant OS (Chimera Linux is the reference platform)
2) A C++17 compiler
3) Meson and Ninja (to build)
4) Dinit (**version 0.16.0 or newer**, older versions will not work)
5) PAM

The system consists of two parts:

1) The daemon `dinit-userservd`
2) The PAM module `pam_dinit_userservd.so`

The PAM module needs to be enabled in your login path. This will differ in
every distribution. Generally you need something like this:

```
session optional pam_dinit_userservd.so
```

The daemon needs to be running as superuser when logins happen. The easiest
way to do so is through a system Dinit service. The project already installs
an example service (which works on Chimera Linux).

## How it works

The `dinit-userservd` daemon manages sessions. A session is a set of logins
of a specific user. Upon first login in a session, the daemon spawns a user
instance of Dinit. Upon last logout in a session, the instance is stopped.
The instance is supervised by the daemon and does not have access to any
of the specific login environment (being shared between logins).

The login will not proceed until all user services have started or until
a timeout has occured (configurable). This user instance will have an
implicit `boot` service, which will wait for all services in the user's
`boot.d` (or another path depending on configuration) to start. If the
`boot.d` does not exist, it will first be created before starting the
user Dinit.

The daemon is notified of logins and logouts through the PAM module. The
daemon opens a control socket upon startup; when a user logs in and the PAM
module kicks in, it opens a connection to this socket and this connection
is kept until the user has logged out. This socket is only accessible to
superuser and uses a simple internal protocol to talk to the PAM module.

The behavior of the daemon is configurable through the `dinit-userservd.conf`
configuration file. The PAM module is not configurable in any way.

Some of the configuration options include debug logging, custom directories
where user services are located and so on. There is also some auxiliary
functionality:

### Rundir management

The daemon relies on the `XDG_RUNTIME_DIR` functionality and exports the env
variable into the service activation environment. The path is specified in
the configuration file and tends to be something like `/run/user/$UID`.

By default, it relies on something else to manage the directory. Typically
this is something like `elogind`.

However, it can also manage the directory by itself, in environments that
do not have anything else to manage it. This is disabled by default and
needs to be manually enabled in the configuration file.

When the daemon manages the directory, the environment variable is also
exported into the login environment in addition to the activation environment.

### D-Bus session bus handling

When using user services to manage your D-Bus session bus, you will have just
one session bus running for all logins of the user, and its socket path will
typically be `$XDG_RUNTIME_DIR/bus`.

By default, if this socket exists by the time the user services have started,
the `DBUS_SESSION_BUS_ADDRESS` environment variable will be exported into
the login environment by the PAM module, pointing to the correct socket.

This can be disabled if desired. Note that if the socket does not exist,
nothing is exported.

This does not take care of exporting the variable into the activation env.
Doing so is up to the user service that spawns the session bus. It can and
should do so with for example `dinitctl setenv`.
