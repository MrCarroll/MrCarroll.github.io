# Fixing the KeepassXC Snap File Browser

## Intro

The version 2.7 release of KeepassXC experienced a regression in the [Snap](https://snapcraft.io/keepassxc) package.
The snap package could not open the GUI to select a file.
Any attempt to trigger the file selection prompt would simply not work, with no visual feedback from KeepassXC itself.
This bug was tracked [here](https://github.com/keepassxreboot/keepassxc/issues/7607).

After some debugging, I'd managed to find the cause of this problem, and thought it may be worth writing down my thoughts.

## The XDG Desktop Portals

The XDG Desktop Portals are a collection of DBus interfaces that are primarily designed around improving system integration with Flatpaks.
However, Snaps equally use these portals; and the portals themselves are becoming heavily utilised even outside of these packaging formats.

The portals provide a means for sandboxed applications to offload a higher privileged request to a trusted intermediary.
One of the most common portals is `OpenFile()`.
This portal provides the illusion that a sandboxed application can see every file on the filesystem.
However, the application will only be given access to the specific file selected.
Further benefits include using the native file chooser dialog for the desktop environment in use, e.g., KDE users will see the KDE file picker even in GTK applications.

Some other common portals include the `ScreenShot()` portal, which provides an abstract API that allows screenshots even on the Wayland display protocol.
Furthermore, the `OpenURI()` portal provides functionality similar to `xdg-open`, and can be used to open web browsers, text editors, etc.
Electron uses the `OpenURI()` portal by default to allow for QT filepickers to function, whereas applications such as OBS Studio use the `ScreenCast` portal to capture the desktop video stream.
They use these portals even outside of Snap/Flatpak environments. 

## KeepassXC 2.6 → 2.7

The 2.7 release of KeepassXC was a big release by itself, but the Snap package had several associated changes.
Firstly, the base snap was bumped from `core18` to `core20`. 
The base contains the most basic execution environment a snap application runs in, and primarily decides the core system libraries (glibc, Python, OpenSSL, etc).
By upgrading from `core18` to `core20`, the snap is effectively rebased from Ubuntu 18.04 to Ubuntu 20.04.

However, the base snap only provides a minimal execution environment.
Desktop snaps in particular will need other libraries, such as Mesa, X11, GTK/QT, etc.
These libraries can be bundled internal to the snap itself, however the latest recommendations in the Snapcraft community is to use the Snapcraft [Extensions](https://snapcraft.io/docs/snapcraft-extensions) system.

KeepassXC 2.6 used the Gnome extension, whereas 2.7 used the KDE Neon extension.

Extensions provide commonly reused collections of libraries, implemented as content snaps; allowing for deduplication of platform libraries.
Additionally, they also take care of other specific concerns, such as setting up the sandboxed environment so that the sound server is functional, or that graphics support works.
The Gnome extension sets up the environment variable `GTK_USE_PORTAL`, which configures GTK based applications to transparently use the XDG Desktop Portals.
However, this variable is not used in the KDE Neon extension.

Upon experimentation, I discovered that the QT libraries were evaluating whether to use the XDG Portals based upon the presence of the `$SNAP` variable.

## Solution 1: Avoid Using the Portals

Since the problem exists because of the use of the Portal API's, one solution was to simply avoid using the portals at all.
Unfortunately, doing this would involve making a wrapper script that removes the `$SNAP` variable prior to execution of KeepassXC.
Whilst this solution works, it's unclean; because the application itself may have uses for that variable beyond setting up the QT variable.

When the Portals aren't being used, the snap would still have access to the users `$HOME` directory through the `home` interface.
However, the portals are still nice to have to fix some integration issues (e.g,. theming, recently used folders, etc).

## Solution 2: Fix the Portals

Looking at the messages being passed on DBUs, using `dbus-monitor`, the following error is shown.

```
error_name=org.freedesktop.DBus.Error.AccessDenied reply_serial=29 string “Portal operation not allowed: Unable to open /proc/5398/root”
```

This error was completely new to me.
Whilst the purpose of the `/proc` directory is well defined, I personally have no real experience with the directory structure and its real world use cases.

My first investigations involved attempting to eliminate any discrepancies that may have occurred from the KDE Neon extension.
This involved me rebuilding the KeepassXC snap with older versions of Snapcraft, modifying the content-snaps manually, etc.
Ultimately, I found it unlikely that the KDE extension was at fault, because other snaps using the KDE extensions were working fine.
Similarly, I determined that there were no **portal** regressions from the Gnome extension to KDE extension change, because upon analysing older versions of the KeepassXC snap, the portals were not being used originally anyway.
Therefore, the bug causing this behaviour likely already existed in older releases but hasn't been noticed until the 2.7 release when the Portals are now being used by default.
After analysing the DBus messages being produced, I decided it was unlikely that the problem existed within the KeepassXC snap itself.
Instead, I assumed that the problems may have been on the Portal implementation side.

Comparing KeepassXC to other snaps, it became clear that there was a difference with the folder `/proc/[pid]/root` itself, namely, this folder was owned by root:root for KeepassXC but owned by the user in every other snap.
Interestingly, even owned by root, the permissions on the folder were 777, i.e., read, write and execute for all users.
This doesn't really make sense, because it was clear that other users couldn't actually access the folder, even if the value for "other" was set to RWX.
I don't have an explanation for this, but presume it may come down to leaky abstractions in the "everything is a file" metaphor in Unix systems.

The `/proc/[pid]/root` shows the host environment the process ID's perspective of the filesystem.
Since strict Snaps and Flatpaks run inside a mount namespace, this folder provides a way for the host system to peek into the mount namespace in use.
My presumption is that the Portals make heavy use of various files/folders in `/proc/[pid]` for the purposes of checking privileges, and that being owned by root breaks their ability to function.

After searching the web for reasons for the folder to be owned by root, the main cause appeared to be relating to the system's crash dump settings.
After attempting to mess around with the various crash dump settings at the system level to no success, I'd noticed references to the `prctl()` system call.
Grepping this function in the KeepassXC revealed it was being used to override the crash dump behaviour at the application level.
After patching this function call out and rebuilding the snap, the portals worked correctly!

In summary, disabling core dumps also has the side effect of making `/proc/[pid]` owned by root, breaking the Portal API's.
Whilst the changes to `/proc/[pid]` are well documented alongside the system call, the system call itself is quite niche, and I believe this is likely the first time it has caused issues publically with the Portals, as the majority of applications would have little reason to invoke this function.

## Analysis
Since KeepassXC is a password manager and held to stringent security standards, it's unfortunate that the snap cannot make use of `prctl()`.
The primary reason KeepassXC would call this function is to prevent the possibility for password/encryption key leakage in the event of a crash.
The system may decide to store a memory dump of the application, and malicious users may take advantage of these files intended to be used for debugging purposes.
However, there are significant benefits to being in the Snap format, so while preventing crash dumps would be a nice to have feature, the other benefits of being in a Snap (e.g,. sandboxing, package verification, etc) weigh heavily.
Particularly, the majority of users are unlikely to be in a scenario where concerns around other users stealing these crash dumps is a practical concern.

It's also likely that this bug would have affected the Flatpak build of KeepassXC, however, I've no idea how QT determines whether to use the XDG Desktop Portals in a Flatpak environment, and my assumption is that currently, the Flatpak does not use these portals or I would have reasonably expected the same bug to occur.

Overall, this was an interesting bug to diagnose and took about eight hours working non-stop, trying different solutions and establishing various ground truths on application behaviour.
It introduced me to some new (perhaps niche) aspects of Linux systems, but importantly, fixed a fairly critical UX impacting huge numbers of people.
Overall, it was time well spent!
