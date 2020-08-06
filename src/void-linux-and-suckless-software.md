# Void Linux and Suckless Software: A Perfect Combo!
<p class="date">August 6, 2020</p>
I'm a huge fan of [Void Linux](https://voidlinux.org/) for many reasons. One of the great things about it is that the distribution is binary focused, yet still provides an elegant tool for compiling software from source: `xbps-src`.

`xbps-src` allows you to build packages from simple templates found in the [void-packages](https://github.com/void-linux/void-packages) repository; unlike the AUR, which creates a huge security risk if you're not reading the PKGBUILD files, templates submitted to void-packages must meet a quality requirement to be included. If a package you want isn't available on the repo, don't fret: you can either create your own template or use someone else's template hosted on their own GitHub account (e.g. the [Liquorix Kernel](https://github.com/DBLouis/void-linux-liquorix)).

One of the other great things about `xbps-src` is that everything is built in a container, offering a huge advantage over manually compiling software. The use of a container environment means that your host system won't be convoluted with unneeded make dependencies, keeping your Linux install lean and minimal.

Finally, my absolute favorite feature of `xbps-src` is that it's incredibly robust. It allows you to easily apply patches and to use custom C header file configs—making `xbps-src` a delight to use with Suckless's tools.

Here, I'll be detailing a step-to-step guide on using `xbps-src` to manage Suckless software.

## Dependencies
`xbps-src` depends on a number of things:

* GNU bash
* xbps >= 0.56
* curl
* flock (provided by util-linux)
* bsdtar or GNU tar
* install (provided by coreutils)
* objcopy, objdump, strip (provided by binutils)
* xbps-uunshare or xbps-uchroot

## Setting Up the Environment
First, you'll have to clone the aforementioned void-packages repository.

```
$ git clone https://github.com/void-linux/void-packages
```
Enter the cloned directory, and setup the initial build container. The set of packages which make up this container is called the `bootstrap`.  

```
$ cd void-packages
$ ./xbps-src binary-bootstrap
```

In the command above, `binary-bootstrap` indicates that container packages will be installed through binaries, either from xbps or a local repository. If you were to substitute `binary-bootstrap` for `bootstrap`, packages will be built from scratch using your host system's toolchain; this is not recommended.

Take note that the above commands only have to be done once to setup `xbps-src`.

## Configuring Packages
This is the fun part, where you get to configure your software with custom configs and patches. We won't have to create our own templates as void-packages already includes most Suckless software.

### Using a Custom config.h
Using a custom config.h file is pretty easy. First, create a "files" directory inside the srcpkg folder of the software you're trying to configure. This folder may/may not already exist; it depends on what the exact package is. 

```
$ mkdir srcpkgs/<package-name>/files
```

In here, we'll put our custom config.h; if you already have a config.h ready to go, just download it and put it into the "files" directory. For example, taking the config.h for my st build (in this use case, <package-name>=st):

```
$ curl https://raw.githubusercontent.com/krishxmatta/st/master/config.h --output srcpkgs/<package-name>/files/config.h
```

If you need the default config.h to modify, follow the below steps:
Extract the package source distribution files into the container. This allows you to see all the source files of the package you're trying to build.

```
$ ./xbps-src extract <package-name>
```

Copy the default config from the package, and place it into the "files" directory of the template.

```
cp masterdir/builddir/<package-name>-<version>/config.def.h srcpkgs/<package-name>/files/config.h
```

You can now edit `srcpkgs/<package-name>/files/config.h` to fit your needs.

### Applying Patches
Applying patches to the software is also a relatively trivial task. First, you'll need to create a "patches" folder in the srcpkg folder of the package you're customizing.

```
$ mkdir srcpkgs/<package-name>/patches
```

Next, you'll have to place the specific patch into the recently created "patches" directory. Using the url to st's `clipboard` patch as an example (in this particular case, <package-name> = st, and <patch-name> = clipboard):

```
$ curl https://st.suckless.org/patches/clipboard/st-clipboard-0.8.3.diff --output srcpkgs/<package-name>/patches/<patch-name>.diff
```

You may have realized that Suckless patches require the argument `-Np1` to be used with `patch`. To achieve this with `xbps-src`, all we have to do is create a file with the exact same name as the patch, but give it the `.args` file extension. This file needs to contain the arguments we will use for patching.

```
$ echo "-Np1" >> srcpkgs/<package-name>/patches/<patch-name>.diff.args
```

You can repeat the above steps for any other patches you'd like to apply to your package. Keep in mind that you'll need a `.args` file for each patch.

## Building and Installing the Package
Finally, once you're satisfied with your tweaks to the package, you'll need to actually compile and install the software. Compiling can be achieved through `xbps-src`, as shown below.

```
$ ./xbps-src pkg -f <package-name>
```

Above, the `pkg` command indicates that we'll be building a binary for the package. The `-f` argument is to force-rebuild the package; this is to ensure that our latest changes will be built.

After the binary has been built, we need to actually install it on our system. We can handle this with `xbps-install` (root permissions are required to run this):

```
$ xbps-install --repository ./hostdir/binpkgs -f <package-name>
```

The `--repository` flag indicates that we'll be installing the package from our local repository, rather than Void's official repositories. The `-f` parameter indicates that we'll be force installing the package, thus wiping out any previous install.

After the package is installed, we have one last step: hold the package. If we were to update our system without holding the package, xbps would install the binary available in Void's official repositories; thus overwriting all of our custom changes. Holding the package prevents this from happening, and can be done like so:

```
$ xbps-pkgdb -m hold <package-name>
```

## Updating Packages
Whenever I use Suckless software, I usually refrain from updating; if it ain't broke, don't fix it. Regardless, if you need a new version for some reason, you'll need to first bring the void-packages repository up-to-date.

```
$ git pull
```

Next, check if your package has been updated.

```
$ xbps-checkvers -D . -I -m <package-name>
```

In the above command, `-D` is used to specify the location of the void-packages directory, `-I` is used to indicate that you want to compare the version of your currently installed package to the latest corresponding template, and `-m` is used to specify which specific package you're checking for.

In the case that your package is outdated, you may have to customize a new config.h to reflect newer changes, and also use updated patches. After doing so, you'll have to rebuild and reinstall the package in the same process shown above.
