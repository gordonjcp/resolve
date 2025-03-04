# Containerizing DaVinci Resolve [Studio] in x86_64 Ubuntu w/NVIDIA

Note:  Please read this in its entirety, including caveats and disclaimers, to understand what you're doing and some of the risks involved.

## What is DaVinci Resolve?

[DaVinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve/) is a popular, powerful free (and paid "Studio" version) [non-linear editor](https://en.wikipedia.org/wiki/Non-linear_editing) for Linux, Mac, and Windows.

DaVinci Resolve was designed to be run on the [now-deprecated](https://thenewstack.io/red-hat-deprecates-linux-centos-in-favor-of-a-streaming-edition/) CentOS Linux distribution.  Ubuntu users who want to run DaVinci Resolve have traditionally [repackaged the official DaVinci Resolve release to a .deb file](https://ubunlog.com/en/makeresolvedeb-facilita-la-instalacion-de-davinci-resolve/) with [a script by Daniel Tufvessons](https://forum.blackmagicdesign.com/viewtopic.php?f=21&t=56878&start=800#p401682) and installed from there.

But running DaVinci Resolve in the "wrong" distribution can reportedly create headaches when updating-- the repackaging script may have to be rewritten to support the new release, there may be library incompatibilities, etc.

There must be another way...

## Let's use a container!

[Containers](https://en.wikipedia.org/wiki/List_of_Linux_containers) allow (among other things) multiple distributions of Linux to run simultaneously on a single machine.  The "host", in this case the Ubuntu installation, runs a containerized version of the "guest" installation, in this case CentOS.  It's a little bit like a [virtual machine](https://en.wikipedia.org/wiki/Virtual_machine), but unlike a virtual machine, the installations of Linux share the same kernel.  Still, the stuff running in CentOS is semi-isolated and usually doesn't know it's running inside another operating system.

So the big idea here is to use Ubuntu as your host computer and put DaVinci Resolve in a CentOS container where it can run in its own native environment with the files-- libraries and such-- it expects.

The goal will be to make it as seamless as possible with the "host" (Ubuntu, in this case) computer.

I'm brand-new to using containers-- and to DaVinci Resolve-- so this will be a bit of an experiment.

Several years ago, [VertexStudio](https://vertexstudio.co/) committed a [method for containerizing DaVinci Resolve to github](https://github.com/VertexStudio/docker-resolve).  Unfortunately, it didn't quite work for me on newer versions of Ubuntu/Resolve.  Still, that earlier repository was very helpful as a reference, so thanks to @rozgo for the great write-up in his [README.md](https://github.com/VertexStudio/docker-resolve/blob/master/README.md), which you should probably look at as well for a bit of context and some more info on why this might be worthwhile, and some more tips.

## So, to sum up-- what's the big advantage of a using a container?

Besides running DaVinci Resolve in its actual intended operating system (CentOS) without ever leaving the comfort of your Ubuntu machine, containers offer some other big advantages:

For one, you can maintain multiple versions of DaVinci Resolve on your system without having to reformat everything and reinstall.  How?  Well, say a new version comes out and you want to test it out-- you can just pop in a new resolve .zip, rebuild a new container image with a single command, and quickly give it a spin-- using your existing projects and media.  If you don't like what you see, you can instantly revert to the previous version (assuming the new version didn't just trash your project or anything, so back up first!)

You can also (theoretically, I haven't tried this) switch between the free and paid version or, hardware allowing, run them both simultaneously-- though maybe not on the same project files at the same time.  That could be nuts.

Containerized, DaVinci Resolve can also be isolated from the Internet while the rest of your computer continues to be connected.  And once the container image is built, it can also be quickly moved onto another machine without having to re-set it all up again.

## Cool, cool.  I'm sold on at least trying this idea.  But which container system to use?  Linux has a few.

Yes, there's lxd and lxc and snap and- y'know, let's narrow it down to two really popular ones-- [Docker](https://www.docker.com/) and [Podman](https://podman.io/).

Docker is by far most popular, but requires a daemon with root access be running all the time.  Hmm.  This could have [security implications](https://mobile.twitter.com/blackmagic_news/status/1470936014646439937?s=21), and may unnecessarily use system resources even when no containers are running.

RedHat has developed an alternative, mostly-Docker-compatible system called the Pod Manager tool, or [Podman](https://podman.io/).  One advantage of Podman is that containers can be run by a single user without ever requiring root.  When running Resolve, we don't ever need root access, so this is my favorite solution.

Both Podman and Docker are available in Ubuntu and can be installed via [apt](https://en.wikipedia.org/wiki/APT_(software)).

## I can't decide... can we make BOTH work?

Sure.  If you have Podman running and prefer that, great.  Or use Docker.  Whatever.

So here's the plan on building the image-- we'll start with the official CentOS 8, then update the packages and add some dependencies needed for DaVinci Resolve.  Y'know, drivers and libraries and stuff.  Then install DaVinci Resolve from the official zip file you can get from the website.  Then create a user called "resolve" in the CentOS container.  That's the user who will run resolve in CentOS.

## Wait- a user named "resolve" will run Resolve?  But _I_ want to run it!  How will I access the projects and stuff?

No problem-- instead of saving data such as our projects and the media IN the Centos container, we'll save it in Ubuntu folders instead. That we can easily interact with it, back it up, and treat it like any other data on Ubuntu.

How, you ask?  We'll map the folders!

## Mapping directories (that is, bind-mounting folders) from CentOS to Ubuntu

Containers like Docker and Podman commonly use "[bind mounts](https://docs.docker.com/storage/bind-mounts/)" to connect a directory in your host Ubuntu account to be accessed from the container as if it was "native" to the container itself.  This gives the container access to certain folders that will persist in your host-- even when the container isn't running.  Why do this?  To retain your projects, cached rendered video, or raw footage-- in **Ubuntu**!  This also allows X11 Xwindows and sound to "pass through" from CentOS to Ubuntu and even provides access to fonts you've installed locally in Ubuntu in the CentOS install of Resolve!

In other words it's cool.

## Sounds cool.

That's what I just said.  In fact, if we map everything right-- the container shouldn't even need to store ANYTHING.  It should just run, and anything we care to save we will save on Ubuntu.  In fact, the container will be "ephemeral", meaning we won't retain anything there-- the stuff we care about will be on the directories on Ubuntu.  We'll start the container fresh every time.

I've set up some basic mounts in this repo-- but feel free to move those mapped folders around to whatever is most convenient.  They're all set up in the `resolve.sh` file used to launch DaVinci Resolve.

The mounted directories will be in `resolve/mounts`.  One special directory in there of note is `resolve/mounts/resolve-home`, which is connected to the "home directory" of the user named `resolve` inside the container.  Again, that's the CentOS user that's actually running DaVinci Resolve.  A nice side-benefit of doing this is that this folder isolates the Resolve-specific user stuff, keeping it separate from your personal Ubuntu home space stuff.  It would be all mingled together if you ran DaVinci Resolve right on Ubuntu.

BTW, in Podman, to enable the files to be accessed by both the host and the container without running into ownership and permissions issues, I do allow the "resolve" container user to access the group name for the host (usually the name of the user).  There's probaby a way to isolate these namespaces completely, but the way I set it up works for me.  If you absolutely don't want the container to know anything about the host for some reason, well, you'll have to [set it up better than I did](https://docs.podman.io/en/latest/markdown/podman-unshare.1.html)!

**A quick note on security in general-- this container setup is _not_ focused on completely isolating the CentOS container from the Ubuntu host-- the goal is mostly to make Resolve work.  Both OSes can share resources and services such as windows integration, USB, keyboard, group names, network IPs, copy/paste etc.  There are nice benefits to using the container like limiting network access, file separation, etc. but this is not designed to be bulletproof.  If for some weird reason you do need the container to be totally and completely isolated from the host, see [here](https://www.redhat.com/en/topics/security/container-security) and [here](https://docs.docker.com/engine/security/).  If you have a commit to address isolation or security concerns without sacrificing functionality or performance, please submit a pull request!**

## USB dongles & Registration Codes

The default startup script `resolve.sh` also maps `/dev/bus/usb` from the Ubuntu host to the CentOS DaVinci Resolve container.  This should allow paid DaVinci Resolve Studio users who use the dongle to have their USB dongles recognized.

If you use Registration Codes instead, there's information below about trying to use them, including the need to enable Internet access to your container (I've turned off Internet access by default).  **Please read the warning about how this is completely untested and you must proceed entirely at your own risk.**

## Speed editor and other dedicated hardware

I also set up the `resolve.sh` startup script to bind mount all the [HID](https://en.wikipedia.org/wiki/Human_interface_device) `/dev/hidraw#` device files to allow the speed editor hardware and I assume other keyboards and input devices to work.  At least it did for me.

Will this work with other specialized editing/camera/etc hardware?  Not sure!

## What you'll need to try this

 - A x86_64 PC (16GB or so.  Blackmagic Design says 32GB RAM minimum for Studio but whatever) with a fairly recent NVIDIA > 4GB GPU + a USB port (if using the Dongle).
 - Ubuntu installed on that computer (I'm using 21.10).  I have not tested on non-GNOME versions of Ubuntu (Xubuntu, Kubuntu, Lubuntu, etc.)
 - Optional:  A USB Dongle to enable the paid DaVinci Resolve Studio
 - Optional:  The DaVinci Resolve Speed Editor (USB only-- see below about BlueTooth operation)
 - Optional:  A registration code-- but again **try this at your own risk/peril!**

## How to set it up

1. On the Ubuntu host, install the latest official proprietary NVIDIA drivers in Ubuntu.  You can do this in the **Software & Updates** app, in the **Additional Drivers** tab.  Reboot and make sure everything works okay.  I have the computer running in "discrete" mode (set in the BIOS).  Not sure if this is needed.  Also, I am logged in using X11 (Xwindows) in the Desktop.  Not sure how well Wayland will work, although it theoretically *should* be compatible.  (You can switch your desktop from Wayland to X11 in Ubuntu's account login screen.)

2. Install Podman [or Docker] and and other dependencies with this command: 
     
    `sudo apt install -y podman fuse-overlayfs nvidia-container-runtime crun`

    (Alternately, you can `apt install -y docker` instead of podman.  But IMO Podman is better/safer.)
 
3.  Move or `git clone` this current repository (with `Dockerfile`, `resolve.sh`, `build.sh`, `.gitignore`, `.dockerignore`, `env-set.sh`, this `README.md`, etc.) somewhere like `~/containers/resolve`, so let's just go with that for now.

4. Download the official DaVinci Resolve `.zip` file from [Blackmagic Design](https://www.blackmagicdesign.com), the makers of DaVinci Resolve.

5. Move that `.zip` file to `~/containers/resolve/`and rename it to be in this format:

      **DaVinci_Resolve_Studio_17.4.2_Linux.zip**

     (You can leave out the **_Studio** part if you're using the free version.)

6. Change directory (`cd`) into `~/containers/resolve` and build the resolve container image:

     `cd ~/containers/resolve`

     `./build.sh`

     **NOTE:  Part of this building/installation process includes agreeing to certain terms and conditions from the makers of DaVinci Resolve [Studio].  Please be sure to review these terms and agree to them before using DaVince Resolve [Studio].  You will be asked to agree when running `./build.sh`.**

7.  Now wait.  The **CentOS 8** system should be downloaded, updated, dependencies added, the DaVinci Resolve.zip copied in there and everything hopefully will be installed.

    Assuming no errors occur, you're (fingers-crossed) ready to run DaVinci Resolve [Studio] now.
    
    NOTE:  In the future, when NVIDIA releases newer drivers, you'll probably want to keep your CentOS container's drivers "in sync" with the one on Ubuntu by rebuilding the container.

8.  To be sure the container's runtime is properly NVIDIA GPU-enabled, try this command:
    
    `./resolve.sh nvidia-smi`
    
    This should output an "information box" from the CentOS container showing the NVIDIA stuff is running successfully there and that it is able to access the GPU.  If it doesn't look right-- whoops.  Not sure what to tell ya, except check out NVIDIA's documentation [here](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) or reddit or maybe the Blackmagic Design forums for suggestions on making it work.
    
    To simplify troubleshooting in this case, you can try a "known working" NVIDIA container rather than the CentOS one you just built.  (Note this involves a pretty big download.)
    
    `sudo docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi`
    
    or
    
    `podman run --rm nvidia/cuda:11.0-base nvidia-smi`
    
    This should output an "info box".  If it doesn't, your problem is probably on the Ubuntu host.  If it does show the box, the problem is probably something about the CentOS container image.

9.  Assuming you're good to go, try running Resolve!

    If you run `resolve.sh` as a regular Ubuntu user with:

    `./resolve.sh`

    ...it MAY just work.  The first time it takes a bit longer as it's setting stuff up.

    Or it may crash.  If it crashes-- see the **Troubleshooting** section below.  You probably just have a few lines to change or something.

     But let's assume for now it worked...

## Make a Dock desktop shortcut

If it worked, you probably don't want to have to type a command every time you want to run it.  Make a desktop shortcut instead!

1.  Find a nice image to use as a dock icon and put it in  `~/.local/share/icons/DV_Resolve.png`  You may be able to copy the desktop image from the running container with this command:
       `podman cp resolve_container:/opt/resolve/graphics/DV_Resolve.png ~/.local/share/icons/DV_Resolve.png`
    (Replace `podman` with `docker` if that's what you're using)
2.  Edit the included `resolve.desktop` file's "Exec" line to run `resolve.sh` in the correct location just as you were typing it (including the full path to `./resolve.sh`) and copy the `resolve.desktop` file to `~/.local/share/applications/resolve.desktop`
3.  In Ubuntu, click the **Show Applications** icon on the launcher or press the the **Super** key (which often has a Windows logo) + A.
4.  Find the DaVinci Resolve icon you created (or you can do a search for "resolve")
5.  Right-click the icon and choose "Add to Favorites"
4.  At this point the icon should appear in your dock.  Click once to launch it.

## Troubleshooting

### I can't move or resize Resolve's main window!  It's locked in place!

Try holding down the **Super** key (which often has a Windows logo).  You should then be able to move or resize the DaVinci Resolve window.  A center-button mouse press should also bring options for minimizing, maximizing, etc.

### The windows are way too small (or too large)!

Play with these lines in `resolve.sh` to adjust how big Resolve appears on your screen:

     --env GDK_SCALE=2 \
     --env GDK_DPI_SCALE=0.5 \
     --env QT_AUTO_SCREEN_SET_FACTOR=0 \
     --env QT_SCALE_FACTOR=2 \
     --env QT_FONT_DPI=96 \

### Why doesn't drag and drop work from Ubuntu?

Because, except for those directories bind-mounted from Ubuntu to the container, the container doesn't know about Ubuntu's desktop-- it's running in CentOS, remember?  You can share screen and sound output and mouse/keyboard/speed editor input, copy/paste text but that's about it.  There may be a way via X11 or GNOME Desktop standards to enable drag and drop with a container, but if so I don't know anything about it.

### Resolve doesn't have network access!

By default, I opted to isolate the container from the Internet.  If you want to give it the same Internet access as your host computer, set an environment variable when running `./resolve.sh` like this:

     RESOLVE_NETWORK="host" ./resolve.sh

This will use the host's network stack.  You can make this behavior persist for the session by setting the environment variable:

     export RESOLVE_NETWORK="host"

And you can add that line to `.bashrc` (or `~/.zshrc`, etc.) or the `resolve.desktop` file so that it is the setting every time.

**NOTE:  If you are enabling the Internet to activate a registration code, read the next bit!**

### Can I use a registration code to activate Resolve Studio in the container?

**IMPORTANT NOTICE:  I HAVE NOT TESTED THE DAVINCI RESOLVE STUDIO REGISTRATION CODES-- ONLY THE DONGLES-- FROM INSIDE A CONTAINER.  I DO NOT KNOW HOW THE REGISTRATION CODES WORKS NOR HOW IT WILL HANDLE CONTAINERS.  TRY THIS ENTIRELY AT YOUR OWN RISK!  I AM NOT RESPONSIBLE FOR LOST/WASTED CODES!**

If you are using Resolve Studio with a registration code, I believe the container should need Internet access so Resolve can contact BlackMagic Design's servers.  I have not tested this functionality in any way, and there may be unforeseen consequences of using a registration code from within a container!

For example-- one way a unique machine can be identified in Linux is by looking at the value of `/etc/machine-id`.  Your Ubuntu host has one `machine-id` value, but it seems weird to pass through the _same_ `machine-id` on a container.  So instead I made it so the CentOS container you create will derive its `machine-id` specifically from your host computer's `machine-id` if it exists (without being identical) and store this derived `machine-id` in your `mounts` directory (named appropriately enough, `container-machine-id`).  Using this _should_ make it so that running newly-built images with updated resolve versions would be consistent at least in terms of the `machine-id`.

Again, I have *no idea* if the `machine-id` is in any way used to identify your computer by Blackmagic Design or if a new image/container might be considered a unique machine and break the activation, so again, USING REGISTRATION CODES WITHIN CONTAINERS IS ENTIRELY AT YOUR OWN RISK AND DO NOT BLAME ANYONE BUT YOURSELF IF THERE ARE ISSUES, INCLUDING THE POSSIBILITY OF LOSING YOUR REGISTRATION CODE(S) ENTIRELY!  PLEASE CONSULT WITH BLACKMAGIC DESIGN SUPPORT TO ASK WHAT THE APPROPRIATE WAY TO MANAGE YOUR ACTIVATIONS SHOULD BE AND WHAT THE POLICIES AND BEHAVIOR ARE FOR CONTAINERIZED INSTALLATIONS OF DAVINCI RESOLVE [STUDIO].

[According to some on Reddit](https://old.reddit.com/r/davinciresolve/comments/rk9k5w/bought_license_awhile_ago_had_to_format_computer/), if you register on a new computer, your older activations will be auto-deactivated.

Hope that's clear.  Post on reddit or something if you have thoughts on this as well as any code adjustments that are needed, which can be submitted as a pull request.

### Does the speed editor work with a *bluetooth* connection rather than USB?

Not yet.  This seems to be an issue not with containers but with either the speed editor firmware or in DaVinci Resolve itself.  Wait for an update I guess.  Becuase it doesn't seem to work yet, this container does not yet install or configure the avahi, dbus, and bluez-hid2hci packages which it might need for future bluetooth support.

### Can I update the speed editor's firmware with this container?

I was able to update the firmware on the Speed Editor from within the container via USB by manually running the Control Panels Setup:

     ./resolve.sh /bin/bash
     
You'll get a shell running in the container.  Now type:

     /opt/resolve/DaVinci\ Control\ Panels\ Setup/DaVinci\ Control\ Panels\ Setup

### What about using postgresql so multiple editors can connect to work on one project?

This script runs the basic single-user installation of DaVinci Resolve with the file-based database, not postgresql.  If you want to use the postgresql database, you'll have to modify the dockerfile to install and configure the database for you and open a port for access.  Should not be too difficult, but perhaps someone else might try it?

### What version of the NVIDIA driver is the container using?

By default it should detect the version YOU were running when you created it.  In the future, you'll probably want to rebuild the container (which again, _shouldn't_ affect the stuff inside your `mounts` directory since that's really on your host computer) when you update your Ubuntu NVIDIA driver so that they match again.  This can be done by simply re-running `build.sh` again.

### How can I poke around the CentOS container from the command line?

As shown above-- instead of `./resolve.sh` try `./resolve.sh /bin/bash` to get a prompt in CentOS.  You can get a root shell (no password needed) by typing `sudo bash`.  (If you want to disable "resolve"'s access to root privileges or change its password from "resolve", you just need to change the `Dockerfile` lines.)

Be advised that changes you make to a non-bind mounted directory inside the container will be *GONE* the next time you run the container, so if you want to keep the changes, bind-mount the directory where you're making changes to somewhere on your host.  This is because by default changes are abandoned and a "fresh" version of the container is run every time.  (In other words, I tried to make it [ephemeral](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#create-ephemeral-containers).)

### Will the container work with other versions of Linux besides Ubuntu?

Dunno.  It probably will work in any distribution that derives from Debian, including Ubuntu, Mint, and others.  However, this theory has not been tested.  So try it and let me know!  Also untested are the various non-GNOME versions of Ubuntu (Xubuntu, Kubuntu, Lubuntu, etc.)  It probably will not work on non x86_64 versions of Linux simply because (as far as I know) DaVinci Resolve is only available for that architecture.

### Where should I put my raw media files or my plugins or new fonts or sound effects libraries etc.?

Fonts you can just add to your local Ubuntu user, but the rest-- that's kinda up to you.  Does Resolve have a standard place for raw media?  As I said, I'm new to Resolve.

The simpliest place for media may be to create a folder in the host's `mounts/resolve-home` (that is, the directory bind-mounted to the home directory of the "resolve" user in the container).  Or you could use the `mounts/Media`, which is bind-mounted to `opt/resolve/Media` in the container.

There's a mount called "BlackMagicDesign/DaVinci Resolve" (that is, `/var/BlackmagicDesign` on the container) that I noticed has folders inside called `Brushes`, `Config` `LUTs`, etc.  Maybe this is where system-wide plug-in stuff goes?  I really don't know enough about Resolve yet to answer this.

As for sound effects libraries, etc-- I guess you can put those anywhere in the bind-mounted areas and then load them into a project (or point to them in the settings).  ...or just add some more `--mount` lines in `resolve.sh` to map any location on your Ubuntu host to somewhere that you want to access it in the container.

From what I'm reading, you can even use `--mount` or --[volume](https://docs.docker.com/storage/volumes) to access remote file servers, USB thumb drives, or some other fancy data store from inside the container.  See the Docker/podman docs for more on how this would work.  I have no clue.

### How do I install the (royalty) free Blackmagic Sound Library?

If you enable the Internet, you should be able to install it right in Resolve, in Fairlight.  However, if you want to download the zip and install it manually without the Internet enabled, here are steps that might work:

First, download the .zip fom Blackmagic Design.  Then, start up the container with a shell:

     ./resolve.sh /bin/bash

Now that the container is running, in the host (Ubuntu) computer, copy the Sound Library zip into the container:

      cp Blackmagic_Fairlight_Sound_Library_Linux.zip resolve_container:/tmp/SL_Linux.zip

Now, in the container, do the installation--

      cd /tmp
      unzip SL_Linux.zip
      chmod u+x Blackmagic_Fairlight_Sound_Library_Linux.run
      mkdir /home/resolve/tmp  # the installer needs a big temporary directory
      TMPDIR=/home/resolve/tmp ./Blackmagic_Fairlight_Sound_Library_Linux.run

This should start up the installer.  Agree to the license and choose the destination for the library.  The default puts it in `/home/resolve/Movies`.  (On your host device, this will translate to `mounts/resolve-home/Movies`)

Be sure to run DaVinci Resolve to make sure the install was a success:

     /opt/resolve/bin/resolve

If it looks good, quit resolve, and then, still in the container, remove that `tmp` directory you created in your home directory.  Double-check the `rm -rf` line as it will fully remove whatever you type next!

     rm -rf /home/resolve/tmp  # get rid of the temporary directory
     exit
 
That should be all there is to it!

### Resolve won't restart!

Maybe there's an old container still running?  You can use `podman ps -a` to check and `podman rm --force resolve_container` to kill a container if it's stuck.  Obviously replace `podman` with `docker` if that's what you're using.

### What's the deal with fonts?  Where are they coming from?

There seems to be a [bug in Resolve](https://forum.blackmagicdesign.com/viewtopic.php?f=21&t=144683) as of at least version 17.2.2 Build 4-- where the normal title generator can find BOTH the system and user-installed fonts, but the Fusion Titles and Title+ [skip some](https://old.reddit.com/r/blackmagicdesign/comments/9x0qzv/text_doesnt_use_system_fonts/). It seemed like the most reliable place for Resolve to find fonts was the container's `/usr/share/fonts`, so I mapped the Ubuntu host's _local_ user fonts (`.local/share/fonts`) to `/usr/share/fonts` in the container-- that way my user fonts will be seen there instead of the container system fonts.

To maximize the number of fonts for Resolve, I was going to also direct Ubuntu's system fonts to the container's local user fonts directory, but holy cow-- there are SO MANY DEFAULT UBUNTU FONTS, including what seemed like an endless list of international fonts, that I ultimately figured the list was too long to be useful for most people.

So I made it an optional flag-- an environment variable called `RESOLVE_ENABLE_HOST_SYSTEM_FONTS`.  Set that to "YES" and you'll get the Ubuntu system fonts.  Skip it and you won't.

To add a local font to your Ubuntu account, just download the `.ttf` from somewhere like [Google Fonts](https://fonts.google.com) and double-click it to start the **Fonts** installer in Ubuntu.  Basically this just copies the .ttf file to `.local/share/fonts`, which you can also do manually to have the fonts picked up by Resolve!

(You can also set the location where fonts are looked for via Fusion Settings in the Fusion menu.)

### Can I put this repository folder somewhere other than `~/containers/resolve`?

It should still work if you put this "resolve" repo in any directory, but in any case, the defaults are configurable.

### Configurable how?

You can put your mounts in one place, your zip in another, and this repository in yet another!

Here are a few environment variables you can set when running `build.sh` and `resolve.sh`:

* `RESOLVE_ZIP` -- Set this to point to your zip _file_.  The default is the repository directory where `build.sh` is located.

* `RESOLVE_MOUNTS_PATH` -- Set this to a _directory_ where you want the `mounts` folder and its contents to be placed.  The default is right in the repository where `resolve.sh` is running.

* `RESOLVE_TAG` -- You can also set the container tag when building _or_ running.  So if you set `RESOLVE_TAG="17.4.3-TESTING"` when building, you'll end up with an image named **resolve/17.4.3-TESTING**.  With `resolve.sh`, setting this variable will specify the tag you want to run.  The default container tag when building is the Resolve version (also tagged "latest").  The default tag for running is "latest".

* `RESOLVE_LICENSE_AGREE` -- set to "Y" or "YES" if you've already previously agreed to the license and don't want to have to answer the question every time you `./build.sh`.

*  `RESOLVE_ENABLE_HOST_SYSTEM_FONTS` -- As mentioned above, you can set this to "YES" to include the Ubuntu host fonts at `/usr/share/fonts` to the container's `.local/share/fonts`.  Doing this however will significantly add many system fonts, many of which you probably won't need.

*  `RESOLVE_NVIDIA_VERSION` -- When building the container image, you can set this to the version of the NVIDIA driver you want to install in the CentOS container.  The default is to match the version number of the NVIDIA driver on the host.

*  `RESOLVE_NETWORK` -- Set to "host" to use the host's Internet/network connectivity.  Other network driver options are described in the [Docker](https://docs.docker.com/network/) and [Podman](https://docs.podman.io/en/latest/markdown/podman-run.1.html) documentation.  The default is "none", meaning the container will not have network access.

These environment variables can be set at the time you run the command, like:

     RESOLVE_TAG="MyTest" RESOLVE_ZIP=/home/myaccount/Downloads/DaVinci_Resolve_Studio_17.4.3_Linux.zip RESOLVE_LICENSE_AGREE="YES" RESOLVE_MOUNTS_PATH="/mnt/myContainers/resolve" ./build.sh
     
or
     
     RESOLVE_ENABLE_HOST_SYSTEM_FONTS="yes" RESOLVE_TAG="MyTest RESOLVE_MOUNTS_PATH="/mnt/myContainers/resolve" ./resolve.sh

You can also set these in advance:

     export RESOLVE_ZIP=/tmp/DaVinci_Resolve_Studio_17.4.3_Linux.zip

Then when you `./build.sh` the next time (at least in this [shell](https://en.wikipedia.org/wiki/Shell_(computing))), it will remember the RESOLVE_ZIP.  To make this more permanent, add to your `~/.bashrc`, `~/.zshrc`, or whatever autoruns when you start a new shell.

### What's next?

There's [a console-based external scripting API](https://forum.blackmagicdesign.com/viewtopic.php?f=21&t=99270), see also [here](https://deric.github.io/DaVinciResolve-API-Docs/), [here](https://diop.github.io/davinci-resolve-api/#/), and [here](https://timlehr.com/python-scripting-in-davinci-resolve/), which can [automate](https://github.com/deric/DaVinciResolve-API-Docs/blob/main/examples/python/3_grade_and_render_all_timelines.py) DaVinci Resolve Studio.  It has been [contemplated](https://forum.blackmagicdesign.com/viewtopic.php?f=21&t=140624), but not AFAIK really explored, how containerized, easily-deployable scripted DaVinci Resolve may be useful, especially for distributed rendering and such.

Apparently, the [$300 Studio version is required for this](https://forum.blackmagicdesign.com/viewtopic.php?f=21&t=77764#wrapper), so someone with multiple licenses, perhaps, can explore further.

### Thanks.

Hope this is helpful to someone.  If so, and you want to give back, consider smashing that like button.. oh wait this is text.  Okay, how about making a US tax-deductable donation to the [Electronic Frontier Foundation](https://www.eff.org)?

### Last thought-- buy this thing!

Based on my very limited experience using DaVinci Resolve, it is EXCELLENT and the community around it and the tutorials I found around, etc are top-notch.  Shout out to [Casey Faris](https://twitter.com/caseyinhd), especially, [on YouTube](https://www.youtube.com/user/CaseyFaris777/videos).  So I do recommend if you enjoy using Resolve, purchase the full "Studio" version, which you can get with the Speed Editor thing.  It really appears to be worth the $300 US.  (And no, no one is paying me to say that.)  Great job, DaVinci/Blackmagic Design!

### Disclaimer

THIS CONTAINER-RELATED SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

DaVinci Resolve, DaVinci Resolve Studio, and Blackmagic Design as well as their owners, officers, employees, shareholders, etc. are not affiliated with and do not endorse this container-related software. DaVinci Resolve, DaVinci Resolve Studio, and the Blackmagic Sound Library are not distributed by the developers of this software and contain their own licenses which must be read and agreed to before they are used.  Please respect their licenses!
