# Flatpak Builder

  If an application requires additional dependencies that aren't provided by its runtime, Flatpak allows them to be bundled as part of the app itself. This requires building each module inside the application build directory, which can be a lot of work. The `flatpak-builder` tool can automate this multi-step build process.

  flatpak-builder takes care of the routine commands used to build an app and any bundled libraries, thus allowing application building to be automated. To do this, it expects modules to be built in a standard manner by following what is called the [Build API](https://github.com/cgwalters/build-api). If any modules don't conform to this API, they will need to be modified.

## Manifests

  The input to flatpak-builder is a json file that describes the parameters for building an app, as well as each of the modules to be bundled. This file is called the manifest. Module sources can be of several types, including .tar or .zip archives, Git or Bzr repositories, patch files or shell commands that are run.

  The GNOME Dictionary manifest is short, because the only module is the application itself:

      {
        "app-id": "org.gnome.Dictionary",
        "runtime": "org.gnome.Platform",
        "runtime-version": "3.22",
        "sdk": "org.gnome.Sdk",
        "command": "gnome-dictionary",
        "finish-args": [ 
           "--socket=x11", 
           "--share=network"  
        ],
        "modules": [
          {
            "name": "gnome-dictionary",
            "sources": [
              {
                "type": "archive",
                "url": "https://download.gnome.org/sources/gnome-dictionary/3.22/gnome-dictionary-3.22.0.tar.xz",
                "sha256": "efb36377d46eff9291d3b8fec37baab2355f9dc8bc7edb791b6a625574716121"
              }
            ]
          }
        ]
      }

## Cleanup

  flatpak-builder performs a cleanup phase after the build, which can be used to remove headers and development docs, among other things. Two properties in the manifest file can be used for this. First, a list of filename patterns can be included:

      "cleanup": [ "/include", "/bin/foo-*", "*.a" ]

  The second cleanup property is a list of commands that are run during the cleanup phase:

      "cleanup-commands": [ "sed s/foo/bar/ /bin/app.sh" ]

  Cleanup properties can be set on a per-module basis, and will then only match filenames that were created by that particular module.

## File renaming

  Files that are exported by a flatpak must be named using the application ID. However, application's source files will typically not follow this convention. To get around this, flatpak-builder allows renaming application icons, desktop files and AppData files as a part of the build process, using the rename-icon, rename-desktop-file and rename-appdata properties.

## Splitting things up

  By default, flatpak-builder splits off translations into a separate .Locale runtime,
  and debuginfo into a .Debug runtime, and adds these 'standard' extension points to
  the application metadata. You can turn this off with the separate-locales and no-debuginfo keys, but there shouldn't be any reason for it.

  When flatpak-builder exports the build into a repository, it automatically includes the .Locale and .Debug runtimes. If you do the exporting manually, don't forget to include them.

## Example

  You can try flatpak-builder for yourself, using the repository that was created in the previous section. To do this, place the manifest json from above into a file called `org.gnome.Dictionary.json` and run the following command:

  
  $ flatpak-builder --repo=repo dictionary2 org.gnome.Dictionary.json
  
  
  This will:

   * Create a new directory (called dictionary2)
   * Download and verify the Dictionary source code
   * Build and install the source code, using the SDK rather than the host system
   * Finish the build, by setting permissions (in this case giving access to X and the network)
   * Export the resulting build to the tutorial repository, which contains the Dictionary app that was previously installed

  flatpak-builder will also do some other useful things, like creating a separately installable debug runtime (called `org.gnome.Dictionary.Debug` in this case) and a separately installable translation runtime (called `org.gnome.Dictionary.Locale`).

  It is now possible to update the installed version of the Dictionary application with the new version that was built and exported by flatpak-builder:

  
  $ flatpak --user update org.gnome.Dictionary
  

  To check that the application has been successfully updated, you can compare the sha256 commit of the installed app with the commit ID that was printed by flatpak-builder:

  
  $ flatpak info org.gnome.Dictionary
  $ flatpak info org.gnome.Dictionary.Locale
  
  
  And finally, you can run the new version of the Dictionary app:

  
  $ flatpak run org.gnome.Dictionary
  
  
## Example manifests

  A complete manifest for [GNOME Dictionary built from Git is available](https://git.gnome.org/browse/gnome-apps-nightly/tree/org.gnome.Dictionary.json)
  , in addition to manifests for [a range of other GNOME applications](https://git.gnome.org/browse/gnome-apps-nightly/tree/).

  ## Working with the Sandbox

  By default, a flatpak has extremely limited access to the host environment. This includes:

   * No access to any host files except the runtime, the app and `~/.var/app/$APPID`. Only the last of these is writable.
   * No access to the network.
   * No access to any device nodes (apart from `/dev/null`, etc).
   * No access to processes outside the sandbox.
   * Limited syscalls.  For instance, apps can't use nonstandard network socket types or ptrace other processes.
   * Limited access to the session D-Bus instance - an app can only own its own name on the bus.
   * No access to host services like X, system D-Bus, or PulseAudio.

  Most applications will need access to some of these resources in order to be useful, and flatpak provides a number of ways to give an application access to them. The build-finish command is the simplest of these. As was seen in a previous example, this can be used to add access to graphics sockets and network resources:

  
  $ flatpak build-finish dictionary2 --socket=x11 --share=network --command=gnome-dictionary
  

  These arguments translate into several properties in the application metadata file:

      [Application]
      name=org.gnome.Dictionary
      runtime=org.gnome.Platform/x86_64/3.22
      sdk=org.gnome.Sdk/x86_64/3.22
      command=gnome-dictionary

      [Context]
      shared=network;
      sockets=x11;

  Note that in this example access to the filesystem wasn't granted. This can be tested by installing the resulting application and running:

  
  $ flatpak run --command=ls org.gnome.Dictionary ~/
  
  
  build-finish allows a whole range of resources to be added to an application. Run `flatpak build-finish --help` to view the full list.

  There are several ways to override the permissions that are set in an application's metadata file. One of these is to override them using flatpak run, which accepts the same parameters as build-finish. For example, this will let the Dictionary application see your home directory:

  
  $ flatpak run --filesystem=home --command=ls org.gnome.Dictionary ~/
  
  
  flatpak run can also be used to permanently override an application's permissions:

  
  $ flatpak --user override --filesystem=home org.gnome.Dictionary
  $ flatpak run --command=ls org.gnome.Dictionary ~/
  
  
  It is also possible to remove permissions using the same method. You can use the following command to see what happens when access to the filesystem is removed, for example:

       $ flatpak run --nofilesystem=home --command=ls org.gnome.Dictionary ~/

## Useful sandbox permissions

  flatpak provides an array of options for controlling sandbox permissions. The following are some of the most useful.

  Grant access to files:

      --filesystem=host    # All files
      --filesystem=home    # Home directory
      --filesystem=home:ro # Home directory, read-only
      --filesystem=/some/dir --filesystem=~/other/dir # Paths
      --filesystem=xdg-download # The XDG download directory
      --nofilesystem=...   # Undo some of the above

  Allow the application to show windows using X11:

      --socket=x11 --share=ipc

  Note: –share=ipc means that the sandbox shares IPC namespace with the host. This is not necessarily required, but without it the X shared memory extension will not work, which is very bad for X performance.

  Allow OpenGL rendering:

      --device=dri

  Allow the application to show windows using Wayland:

      --socket=wayland

  Allow the application to play sounds using PulseAudio:

      --socket=pulseaudio

  Allow the application to access the network:

      --share=network

  Note: giving network access also grants access to all host services listening on abstract Unix sockets (due to how network namespaces work), and these have no permission checks. This unfortunately affects e.g. the X server and the session bus which listens to abstract sockets by default. A secure distribution should disable these and just use regular sockets.

  Allow the application talk to a named service on the session bus:

      --talk-name=org.freedesktop.secrets

  Allow the application talk to a named service on the system bus:

      --system-talk-name=org.freedesktop.GeoClue2

  Allow the application unlimited access to all of D-Bus:

      --socket=system-bus --socket=session-bus

## Distributing Applications

  As has already been seen, flatpak installs runtimes and apps from repositories. To do this, it uses [OSTree](https://ostree.readthedocs.io/en/latest/). This is similar to Git, but has been designed to handle trees of large binaries. Like Git, it has the concept of repositories and commits. Applications are stored as branches.

  To distribute an application, it must be exported to a repository. This is done using the build-export command:

  
  $ flatpak build-export [OPTION…] LOCATION DIRECTORY [BRANCH]
  
  
  The resulting repository is in an archive-z2 format. To allow users to use a repository, all you have to do is copy it to a web server and give them the URL.

## Managing repositories

  The flatpak build-update-repo command provides most of the tools for managing repositories. For example, to set a user readable name for a repository:

  
  $ flatpak build-update-repo --title="Nice name" repo
  
  
  build-update also lets you prune (`--prune`) unused objects and deltas from the repository, and even remove older revisions (using `--prune-depth`) which is useful for things like automatic nightly build repositories.

## AppData

  As already described, flatpak uses the AppData standard to store user visible information about applications. This information needs to be accessible to clients in order to be displayed in app stores. To do this, build-update-repo scans all the branches in the repository for AppData data, which is collected and committed into a repository-wide AppStream branch. flatpak then keeps a local copy of this branch for each remote, which can be manually updated using the update command. For example:

  
  $ flatpak update --appstream gnome
  
 
