# Flatpak

Welcome to Flatpak's developer documentation! This covers everything you need to know to build and distribute applications as Flatpaks. The docs start with introduction to the basic concepts and a simple app building tutorial, before moving on to cover automated building and repository hosting.

Example tutorials are used throughout. To complete them, it is necessary to have flatpak and flatpak-builder installed on your system. [flatpak.org](http://flatpak.org/getting.html) has details on how to do this.

## Key Concepts

Flatpak is best understood through its key concepts: runtimes, bundled libraries, SDKs and sandboxes. These help to explain how Flatpak differs from traditional application distribution on Linux, as well as the framework's capabilities.

### Runtimes<a id="runtimes"></a>

Runtimes provide the environment that each application runs in, including the basic dependencies they might require. Each runtime can be thought of as a `/usr` filesystem (indeed, when an app is run, its runtime is mounted at `/usr`). Various runtimes are available, from more minimal (but more stable) Freedesktop runtimes, to larger runtimes produced by desktops like GNOME or KDE. (The [runtimes page](runtimes.html) provides an overview of the runtimes that are currently available.)

Each application must be built against a runtime, and this runtime must be installed on a host system in order for the application to run. Users can install multiple different runtimes at the same time, including different versions of the same runtime.

Flatpak identifies runtimes (as well as SDKs and applications) by a triple of name/arch/branch. The name is expected to be in inverse-dns notation, which needs to match the D-Bus name used for the application. For example: `org.gnome.Sdk/x86_64/3.14` or `org.gnome.Builder/i386/master`.

### Bundled Libraries<a id="bundledlibraries"></a>

If an application requires any dependencies that aren't in its runtime, they can be bundled along with the application itself. This allows apps to use dependencies that aren't available in a distribution, or to use a different version of a dependency from the one that's installed on the host.

Both runtimes and app bundles can be installed per-user and system-wide.

### SDKs (Software Developer Kits)<a id="sdks"></a>

  An SDK is a runtime that includes the 'devel' parts which are not needed at runtime, such as build and packaging tools, header files, compilers and debuggers. Each application is built against an SDK, which is typically paired with a runtime (this is the runtime that will be used by the application at runtime).

### Sandboxes<a id="sandboxes"></a>

  With Flatpak, each app is built and run in an isolated environment. By default, the application can only 'see' itself and its runtime. Access to user files, network, graphics sockets, subsystems on the bus and devices have to be explicitly granted. (As will be described later, Flatpak provides several ways to do this.) Access to other things, such as other processes, is (deliberately) not possible.

## Technologies

  Flatpak tries to avoid reinventing the wheel. We build on existing technologies where it makes sense. Many of the important ingredients for Flatpak are inherited from Linux containers and related initiatives:

  * The [bubblewrap](https://github.com/projectatomic/bubblewrap) utility from [Project Atomic](http://www.projectatomic.io/), which lets unprivileged users set up and run containers, using kernel features such as
   * Cgroups
   * Namespaces
   * Bind mounts
   * Seccomp rules
  * [systemd](https://www.freedesktop.org/wiki/Software/systemd/) to set up cgroups for our sandbox
  * [D-Bus](https://www.freedesktop.org/wiki/Software/dbus/), a well-established way to provide high-level APIs to application
  * The OCI format from the [Open Container Initiative](https://www.opencontainers.org/), as a convenient transport format for single-file bundles
  * The [OSTree](https://ostree.readthedocs.io/en/latest/) system for versioning and distributing filesystem trees
  * [Appstream](https://www.freedesktop.org/software/appstream/docs/) metadata that makes Flatpak apps show up nicely in software-center applications

## The flatpak Command

  `flatpak` is the tool that is used to install, remove and update runtimes and applications. It can also be used to view what is currently installed, and has commands for building and distributing application bundles. `flatpak --help` provides a full list of available commands.

  Most `flatpak` commands are performed system-wide by default. To perform a command for the current user only, use the `--user` option.

## Anatomy of a Flatpak App

  Each Flatpak app has the following basic structure:

      metadata    # A keyfile which provides information about the application,
                  # including information that is necessary for setting up the sandbox
                  # for running the application
      /files      # The files that make up the application
      /files/bin  # Application binaries
      /export     # Files which the host environment needs access to, such as the
                  # application's desktop file, icon and D-Bus service file

  A typical metadata file looks like this:

      [Application]
      name=org.gnome.gedit
      runtime=org.gnome.Platform/x86_64/3.22
      sdk=org.gnome.Sdk/x86_64/3.22
      command=gedit

      [Context]
      shared=ipc;network;
      sockets=x11;wayland;pulseaudio;
      devices=dri;
      filesystems=host;

      [Environment]
      GEDIT_FOO=bar

      [Session Bus Policy]
      org.extra.name=talk
      org.other.name=own


  This specifies the name of the application, the runtime it requires, the SDK that it is built against and the command used to run it. It also specifies file and device access, sets certain environment variables (inside the sandbox, of course), and how to connect to the session bus.

  All the files in the export directory must have the application id as a prefix. This guarantees that applications cannot cause conflicts, and that they can’t override any system installed applications.

### AppData

  Many Linux distributions provide an app store or app center for browsing and installing applications. [AppData](https://www.freedesktop.org/software/appstream/docs/chap-Quickstart.html#sect-Quickstart-DesktopApps) is a standard format for providing application information that can be used by app stores, such as an application description and screenshots. Flatpak makes use of the AppData standard, and application authors are recommended to use it to include information about their applications.

### Extensions

  Applications and runtimes can define extension points, where optional pieces can be plugged into the filesystem. Flatpak is using this to separate translations and debuginfo from the main application, and to include certain parts that are provided separately, such as GL libraries or gstreamer plugins.

  When flatpak is setting up a sandbox, it is looking for extension points that are declared in the application and runtime metadata, and mounts runtimes with a matching name. A typical extension section in a metadata file looks like this:

      [Extension org.gnome.Platform.GL]
      version=1.4
      directory=lib/GL

  More complicated extension points can accept multiple extensions that get mounted
  below a single directory. For example, the gstreamer extension:

      [Extension org.freedesktop.Platform.GStreamer]
      version=1.4
      directory=lib/extensions/gstreamer-1.0
      subdirectories=true

  The subdirectories=true key instructs flatpak to mount e.g. a org.freedesktop.Platform.GStreamer.mp3 runtime on /usr/lib/extensions/gstreamer-1.0/mp3 in the sandbox. The gstreamer libraries in the org.freedesktop.Platform runtime have been configured to look in this place for plugins.

