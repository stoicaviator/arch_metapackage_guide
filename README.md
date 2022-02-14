# A guide to using meta-packages with Arch Linux.

## Disclaimer
I am not an expert, nor am I an Arch Linux Dev or Trusted User. What follows is my best understanding based on the resources available. If you attempt anything described here on a live system without backups and lose your data, that's on you. Make backups and understand what commands do before you type them in.

## Introduction
This guide is intended to give Arch Linux users who are new to the idea of meta-packages and self-hosted repositories (repos) the basics of maintaining their own packages and repos. This topic is immensely useful for anyone that has more than one computer and would like to sync package changes (not configuration changes) across them. The information is wholly available in the Arch wiki, but does not currently exist compiled in a single page. As I'm familiar with GitHub formatting, the information was easier for me to compile here. It is my intention to port some (or all) of this over to the Arch Wiki.

This guide has 6 main parts. Basics, Repositories, Meta-packages, Recommendations, Conclusions and Bibliography. Basics is laid out in an FAQ style ensuring a fundamental understanding of the components we are going to be working with. Repositories will talk about how to create two types of self-hosted repositories: as a directory on your local machine, as well as a simple repository hosted over your network using a simple webserver (though this could be substituted for any other protocol: samba, nfs, ftp, etc). Meta-packages describes how to create simple meta-packages starting with a PKGBUILD template and ending with adding them to a self-hosted repository. Recommendations contains a few tips and tricks that I have found work best for me in maintaining my own self-hosted repositories with my own packages and meta-packages. Conclusions is just some closing remarks. Bibliography will link to all the wiki pages containing information I sourced to compile this document. Additional information may have been extracted from man pages.

While this may seem complicated and like it's a lot of information, building your own meta-packages and hosting your own repository is actually very simple at its heart thanks to how Arch Linux works. Many people who struggle with installing Arch Linux are reminded that it's designed to be "user-centric, not user-friendly" and aims to simplify development, not be easy to use. Once you understand how to create your own packages and host your own repositories, you'll learn very quickly just how true those statements are.
I hope you find this guide useful and the experience of creating your own packages and hosting your own repositories to be rewarding.

## A note on conventions
I've tried to be as generic as possible, but in the name of accuracy and to ease testing, some of the examples are very specific, especially when it comes to naming. I am making an assumption that the user has some basic understanding of the Linux filesystem and basic Linux operations. I also initially tried to explain when there were many options available, but eventually gave up. So as a general disclaimer, be aware that some things need not be exactly as specified by me here. For example, I talk about package files and database files as being tar files, optionally compressed. I personally do use compression and it's usually configured by default. However, the type of compression may differ. Where I have the `.gz` extension, you may have `.xz` or some other compression extension. In general these are interchangeable. If I find a situation where it isn't, I will specify. This may apply to other things as well.

### Table of Contents
- [Basics](#basics)
  - [Q1. What is a package?](#q1-what-is-a-package)
  - [Q2. What is a PKGBUILD?](#q2-what-is-a-pkgbuild)
  - [Q3. What is the Arch Build System?](#q3-what-is-the-arch-build-system)
  - [Q4. What is a dependency? Also, what does it mean to say a package is installed 'explicitly'?](#q4-what-is-a-dependency-also-what-does-it-mean-to-say-a-package-is-installed-explicitly)
  - [Q5. What is a meta-package?](#q5-what-is-a-meta-package)
  - [Q6. What is a group?](#q6-what-is-a-group)
  - [Q7. What is a repository exactly?](#q7-what-is-a-repository-exactly)
  - [Q8. What is the AUR? How do AUR helpers work?](#q8-what-is-the-aur-how-do-aur-helpers-work)
  - [Q9. What do you mean by "local repo" and "hosted repo"?](#q9-what-do-you-mean-by-local-repo-and-hosted-repo)
- [Repositories](#repositories)
  - [Manually Build an AUR Package](#manually-build-an-aur-package)
  - [Pacman and Repos](#pacman-and-repos)
  - [SigLevel and Package Signing](#siglevel-and-package-signing)
  - [Custom Local Repos](#custom-local-repos)
    - [Local Repo Creation](#local-repo-creation)
    - [Local Repo Configuration](#local-repo-configuration)
    - [Local Repo Usage](#local-repo-usage)
  - [Custom Hosted Repos](#custom-hosted-repos)
    - [Webserver Installation](#webserver-installation)
    - [Hosted Repo Creation](#hosted-repo-creation)
    - [Hosted Repo Configuration](#hosted-repo-configuration)
    - [Hosted Repo Usage](#hosted-repo-usage)
- [Meta-Packages](#meta-packages)
  - [Meta-Package Template](#meta-package-template)
  - [Create a Custom Install Meta-Package](#create-a-custom-install-meta-package)
  - [Use our Custom Meta-Package](#use-our-custom-meta-package)
  - [Modify our Custom Meta-Package](#modify-our-custom-meta-package)
  - [Integrate AUR into our Custom Meta-Package](#integrate-aur-into-custom-meta-package)
- [Recommendations](#recommendations)
  - [Use Aurutils](#use-aurutils)
  - [Host AUR Packages](#host-aur-packages)
  - [Use a Version Control System](#use-a-version-control-system)
  - [Consider Security](#consider-security)
- [Conclusions](#conclusions)
- [Bibliography](#bibliography)
 
## Basics
### Q1. What is a package?
A package is a single compressed archive that contains all the information and files necessary for the Arch Linux **pac**kage **man**ager (pacman) to install a **single**, specific application. For example, the package for the default shell `bash` is a compressed archive containing all the information about `bash` as well as the `bash` executable, a host of configuration files and the manual pages. The idea is that any computer containing all the pre-requisites for a given application (dependencies, architecture, pacman, etc) could, with only the package compressed archive, install the complete application. This is true for a small, simple application with one executable file and no additional config files or data as well as for a complex application with a bunch of files like LibreOffice.

You can identify package files easily because they have the `pkg.tar` extension (often compressed, adding a compression extension onto the end of that, for example `pkg.tar.gz`). Its important to note that all packages in Arch Linux are binary packages, meaning no source is downloaded or compiled when you install a package. For those who are just learning, you may think this statement is incorrect because you've installed packages that used git source, but if you read on you will see where that confusion comes from. All the files hosted on the official Arch Linux repos (core, community, etc) are packages.

### Q2. What is a PKGBUILD?
A PKGBUILD is a file used by developers to guide the Arch Build System (see [Q3](#q3-what-is-the-arch-build-system)) on creating a package from sources. It contains information like what version the build package will be, which the build package's description is, what files will be included in the built package, etc. It also contains any instructions that may be required on where to install files, or scripts to run at various parts of the install, among other things. If a PKGBUILD is placed in a directory with all the required source files (or links to the correct location of the source files), invoking the Arch Build System will result in a single package file in the form of a (most likely compressed) archive as described above, ready for distribution and installation to other users or systems.

### Q3. What is the Arch Build System?
The Arch Build System (referred to as the ABS), is the framework by which the Arch Linux development tools create packages from simple, easy-to-read instructions. Some packages require a lot of files to be included and dozens of libraries to be installed, others are straightforward. It is a detailed, robust, extremely adaptable system that takes a lot of diligent study to master. We will only examine a small portion of the Arch Build System in this guide.

### Q4. What is a dependency? Also, what does it mean to say a package is installed 'explicitly'?
There are two ways a package can be installed: explicitly or as a dependency. A package is installed explicitly when you directly tell pacman to install it. For example, `pacman -S base`. `base` is being passed explicitly to pacman. `base` needs some other packages to work properly too though. For example, it needs `filesystem`, which is the package that contains all the files that are required for Arch Linux to work that aren't part of other packages. A good example of this is `/etc/fstab`; it's not installed by any other package or application and yet Arch Linux needs this file to run, so it is included in the `filesystem` package. This requirement for `base` that `filesystem` needs to be installed is called a dependency. `base` depends on `filesystem` to work, so `filesystem` is a dependency of `base`. 

As you might guess, `base` depends on a lot of things. And some of those things depend on other things. Much like a family tree, you can imagine how this system of dependencies might spread out, and this entire family of dependencies is called a dependency tree. Just as `base` depends on `filesystem`, `filesystem` depends on a package called `iana-etc`. So when you install `base`, first pacman installs `iana-etc` so that it can install `filesystem`, then it installs `filesystem` so that it can install `base`, and then it installs `base`. Of course, it also has to install all of the other dependencies to install `base` and they may have their own dependencies and so on, but you get the gist. A single explicitly-installed package could result in a huge dependency tree being installed to support it.

When pacman installs a package, it stores some simple information about it in its own local database on your computer. It does this to keep track of what packages are there, which is why when you run a system update with `pacman -Syu` it knows which packages you have: it has them all listed in a database. It stores a bunch of information, but for the purpose of this guide we need to know it stores the package name, the package version, the package release number and whether it was installed explicitly or as a dependency.

The reason to differentiate between explicit or as a dependency is because dependencies change. Imagine package `foo` depends on package `bar`. But a few years down the road something much better than `bar` comes out called `baz`, and `foo` stops needing `bar` but needs `baz` instead. Once we update `foo`, `bar` would still be installed as a dependency, but nothing would need it. We would now call `bar` an orphan - it was installed as a dependency to a package that no longer depends on it. Unless someone has made an error somewhere, orphans can be safely removed from your system, and many Arch users check for orphans on a regular basis.

### Q5. What is a meta-package?
A meta-package is a term for a specific type of package that actually contains no applications or files itself, but rather serves as an umbrella that encompasses a number of packages underneath it. In essence, a meta-package is a package that is completely empty except that it depends on other packages. This simplifies the installation (or removal) of multiple packages by installing them all automatically when the meta-package is installed. It also serves to make the list of explicitly installed packages smaller. For example, if we had a meta-package called `foobarbaz` consisting of the packages `foo`, `bar` and `baz`, we could install them all simply with `pacman -S foobarbaz`. If we output a list of explicitly installed packages, we would see `foobarbaz` in the list, but not `foo`, `bar`, or `baz`. We could see them all if we output a list of all installed packages, or packages installed as a dependency. We could also uninstall them all with `pacman -Rs foobarbaz`.

There is an important distinction here. Remember I said a package was a single compressed archive that contains all the information and files necessary for pacman to install a specific application? In this case, we have no files. The package `foobarbaz` **does not contain** the files for `foo`, `bar` or `baz`, only instructions to pacman that they are dependencies. So if you took your `foobarbaz` package file to a computer with no internet access and tried to install it, pacman would complain about missing dependencies `foo`, `bar` and `baz` unless you had already installed them before. The `foobarbaz` package tells pacman it needs `foo`, `bar` and `baz`, but it does not contain their files. It may sound like I'm repeating myself, but I have met many people who assume that a package contains its dependencies.

Some users have complained about this. But if you think about it for a second, it only makes sense. Let's just show how unrealistic it is to expect packages to have their dependencies bundled in. Let's imagine you want to install the open-source version of Visual Studios. This package is called `code`. `code` depends on 13 other packages, most of which have their own dependencies. Several of them depend on `libx11`. `libx11` has 4 dependencies, some of which are shared with dependencies of these 13 other packages too. And if you drill down, you would find dozens of these packages depend on `filesystem`, which as we know depends on `iana-etc`. That means you'd have dozens of copies of `iana-etc` bundled inside packages, which are bundled inside other packages, and bundled inside yet more packages, which are bundled in with `code`. So now every single package you try to install requires hundreds of MiB of downloads. That's not just time you spend downloading (and disk space), but internet bandwidth for you, internet bandwidth for the Arch servers, load on the Arch servers, etc. It is unreasonable to expect or demand that packages have their dependencies bundled in with them, as convenient as that may be in very specific cases.

### Q6. What is a group?
A package group is like a meta-package, except that all members of the group are installed explicitly. Remember a meta-package only has dependencies, and while the meta-package and its dependencies all get installed by pacman, only the meta-package is installed explicitly. With a group, you don't end up with a package named after the group installed explicitly and a bunch of installed dependencies, you end up with all members of the group installed explicitly. Using our `foo`, `bar` and `baz` packages, let's make an example. Imagine have a meta-package called `foobarbaz-meta` and a group called `foobarbaz-group`. If we installed the meta-package with `pacman -S foobarbaz-meta`, and then checked a list of installed packages, we would see `foobarbaz-meta` installed explicitly, as well as `foo`, `bar`, and `baz`installed as dependencies. If instead we installed the group with `pacman -S foobarbaz-group`, and then checked a list of installed packages, we would see `foo`, `bar` and `baz` installed explicitly, but no mention anywhere of anything called `foobarbaz-group`.

### Q7. What is a repository exactly?
A repository is, in essence a directory containing a database file which lists all the packages stored in the repository and what their latest version is, as well as the package files themselves. That's really all there is to it. There is no mystery to a repository and they are not difficult to create or host.

### Q8. What is the AUR? How do AUR helpers work?
The Arch User Repository (AUR) is not really a repository as we've discussed them. As we've already covered, a repository is a directory containing a database and a bunch of packages, and packages are self-contained compressed archives containing all the information and files required to install a particular package. The AUR does have a database, but that database is not the same as the repository databases we'll be dealing with. It also doesn't contain packages, it contains what are referred to as "package snapshots" (which I will just call "snapshots" from here on). These snapshots contain the source files required by the ABS to build packages. While commonly referred to as "AUR packages", they are in fact not packages until after you download the snapshot and turn them into a package using the ABS. Even if you use an AUR helper now, such as `yay`, your first install required you to make `yay` yourself by downloading the snapshot from the AUR and using the ABS to create a `yay` package, which you then installed with pacman. We will be covering how to do this a bit later if you have forgotten how.

AUR helpers work by abstracting a lot of this away. The most basic category of AUR helper is a snapshot downloader. The most popular of these is probably `auracle-git`, but very few users use an AUR helper that doesn't go beyond being a snapshot downloader. It's also worth noting `auracle-git` and many other snapshot downloaders offer other useful features as well, but they aren't really germane to this discussion. They essentially download the AUR package snapshot for you, and then it's up to you to build and install the package. 

The next category is the package builder. The most popular of these, and my AUR helper of choice, is `aurutils`. These types of AUR helpers also download snapshots, but they also build packages from them. Depending on how they are designed, they may leave you with just the package files (the `*.pkg.tar.gz` files built using ABS) or they may add them to a local repository.

The last type of AUR helper is a pacman wrapper. The most common of these is `yay`. They also download snapshots, and they also build them into packages, but they go one step further and call pacman to install them for you as well. It's these types of AUR helpers that contribute to the misunderstanding that the AUR is hosting "packages", and that cause so much confusion about what a package is. When you try to install a package using `yay`, you have no idea if it's downloading source from git using an AUR snapshot, and you have no idea what the resultant package is like.

This is why the AUR Helpers wiki page has the header, in a big read box, that says: "Warning: AUR helpers are not supported by Arch Linux. You should become familiar with the manual build process in order to be prepared to troubleshoot problems." If every Arch user had to install at least one application from the AUR that depended on several other applications from the AUR manually, the fact that a build package does not contain its dependencies would be abundantly clear.

AUR helpers are certainly the perfect example of how something user-friendly can obfuscate what's happening in the background so much that it misleads users into a complete misrepresentation of reality. There is no doubt they are convenient, but there is a reason "the Arch way" is no AUR helper at all. This again demonstrates how Arch is not user-friendly, but instead user-centric. The ABS and the AUR are built around users needs, not to cater to making it easy.

### Q9. What do you mean by "local repo" and "hosted repo"?
In theory, what I call a "hosted repo" is incorrect. I don't mean we want to have our repo hosted on AWS or anything that complex, I simply use the term "local repo" to mean "installed on the local computer" and "hosted repo" to mean "installed on one of our computers on the network and available over the network to other computers on the network." It's a convention I only devised for use in this guide to differentiate the two types of repos I wanted to cover. While I will be using a webserver in my examples, it is worth noting that this isn't the only way. You could use FTP, Samba, NFS or something else to share your repo over your network, but I find lighttpd is such an easy webserver to set up and use that I decided to use it in my examples. So just bear in mind these terms aren't official or universal, and are conventions just for this guide.

## Repositories
In this section we will set up and learn how to use our own repositories. This is where the power of not just the ABS, but the modular design of pacman itself really shines. Once you realize how simple it is to create a local repo and host your AUR packages there, you'll realize the abstraction of the more advanced AUR helpers didn't actually save you all that much.

### Manually Build an AUR Package
We're going to need a package to put in our repo once we get it setup, and with the popularity of pacman wrapper AUR helpers (from the now-defunct `yaourt` and `pacaur` to the current reigning champs `yay` and `paru`), I thought it would be a good idea to review how to manually build and install an AUR package. I'm going to use `aurutils`, since I am going to be recommending it later anyway. Building a package from the AUR manually is a 5 step process:

1. Find the application you want and the snapshot URL
2. Download the snapshot
3. Unpack the snapshot
4. Review the PKGBUILD at a minimum, even better to verify the scripts are in order
5. Build the package from the snapshot using the ABS

Complication may arise at step 5 if the package has dependencies that are also on the AUR. In that case, you have to go back to Step 1 for each dependency, which may introduce more dependencies. While doing this at least once is a good exercise to learn how to build packages and see how packages work on Arch, it's also the reason some sort of AUR helper eventually becomes a must have. Note that you might also need to import a key if the maintainer of your AUR package, which is done with `gpg --recv-keys` followed by the key in question. In the case of `aurutils`, you need AladW's key if you don't already have it, so you would use `gpg --recv-key 6BC26A17B9B7018A` to download it and put it in your keyring.

Luckily for me, or perhaps it's good planning, all of the dependencies for `aurutils` are in the official repos, so I will have no dependencies to resolve. So first I go to the AUR website (https://aur.archlinux.org) and I search for aurutils. Then, because I prefer the console, what I do is I right-click on the "Download Snapshot" link and copy it to my clipboard. You could always just click the link and let the browser download it, but that's not my style. So let's build `aurutils`:
```bash
# Download the snapshot
% wget https://aur.archlinux.org/cgit/aur.git/snapshot/aurutils.tar.gz
# Unpack the snapshot
% tar xvf aurutils.tar.gz
# I like a clean system so delete the compressed archive
% rm aurutils.tar.gz
# Review the PKGBUILD
% cd aurutils
% vim PKGBUILD
# Build the package, syncing dependencies
% makepkg -s
```
We should now have a built package for aurutils. You'll recognize it because it will take the format of `aurutils-x.x.x-y.pkg.tar.gz`, where x.x.x is the version number (`pkgver` from the PKGBUILD) and y is the release number (`pkgrel` from the PKGBUILD). If we wanted to install it we could use `pacman -U aurutils-*.pkg.tar.gz` or `makepkg -i` (which just calls the previous pacman command). Alternatively, you can build your package using `makepkg -si` and it will sync dependencies (essentially `pacman -S --asdeps ${depends[@]}` where ${depends[@]} is the `depends` array from the PKGBUILD), and then install the package. As you are noticing, pacman is still doing all the heavy lifting.

### Pacman and Repos
We know what a repo is, so now let's look at how they are used by pacman. Information on repos are stored in the file `/etc/pacman.conf`. Anytime you see something in square brackets, you're seeing a repo configuration block (with one exception being the specially defined `[options]` block that must exist and which defines global settings for pacman). For example:
```bash
[core]
Include = /etc/pacman.d/mirrorlist
```
This is the configuration block that defines the `core` repo, where important packages like `base`, `bash` and `git` live. The first line, `[core]`, tells pacman, "There is a repo named `core` and what follows is the information about that repo." `Include = ` directives are very common in programming and SysOps/DevOps though the syntax differs from place to place, but it is basically telling a program parsing it "when you are reading this, place a copy of this file referenced here at this location." So if `mirrorlist` only contained the word "oops", your pacman would interpret your `core` repo configuration as follows:
```bash
[core]
oops
```
Obviously you would get an error if this was the case and you ever tried to use pacman. What happens instead though is pacman actually goes to your mirrorlist file, finds a URL to a mirror as the `Server =` entries that are uncommented, and uses those as the location(s) of the repo. As discussed previously, that means there is a database file containing a list of all the packages in that repo as well as their version numbers, and also all the package files for those packages at that location. The database file must be named the same as the repo, it must be a tar file (may also be compressed if you'd like) and must have an extension of `.db.tar` or `.files.tar` (and will have the further extension of the compression method used, if one was, for example `.db.tar.gz`). 

So for our `core` repo, if you went to one of your mirrors, you'd see that at the specified location is a huge list of package files (ending in `.pkg.tar.gz`) with some accompanying `.sig` files for verification, as well as the repo database in a few different formats: `core.db`, `core.db.tar`, `core.db.tar.gz`, `core.files`, `core.files.tar`, etc). Aside from the `.sig` files, this is all stuff we already knew about. For reference, the `.sig` files are cryptographic files that prove a package file was created by one of the Arch developers and hasn't been tampered with. Let's take a moment to discuss those.

### SigLevel and Package Signing
Package signing is crucial to ensuring the integrity of core packages. Without package signing, there would be no way to know that the packages created by the Arch devs that we are installing haven't been tampered with. As a result, the security of any Arch Linux system would be called into question. When packages are created by the Dev team, they include a `.sig` file that contains the cryptographic proof that it was indeed built by who it says it was, and that the package file itself has not been tampered with. This package signing is a requirement of the development team, but it's an optional setting in pacman. By default, all packages installed by pacman **require** that they have been signed. The package database (as opposed to the individual packages), however, does not currently require this by default and the official repo databases are not signed at this time anyway (it's a work in progress, to be rolled out eventually).

To confirm pacman is indeed using the default package signing settings, check your `/etc/pacman.conf` file again, looking in the `[options]` section we spoke about for the setting `SigLevel =`. There are 3 options: Never, Optional, and Required. Never means packages are never checked to make sure they are signed and that the signature data indicates they are authenticate and haven't been tampered with, even if the data is available. Optional means check it when data is available, but if packages aren't signed just go ahead and assume they are fine. And Required means check all packages and only install those that are signed and whose signatures are valid. 

Each option can also be specified to apply only to Package or Database. For example, `PackageRequired DatabaseNever` would mean you want all packages you install to be checked, but don't check databases even if signing information is available. Your default SigLevel setting should be `Required DatabaseOptional`, which is saying "I want signature checking to be required globally for everything, but for databases specifically, it's optional." Currently, since there are only 2 options, this would be the same as `PackageRequired DatabaseOptional` but takes less space.

There is one other setting that can go in SigLevel, and that is to what degree you want signatures checked. One option, the default option which is why it's never specified, is `TrustedOnly` which will only accept signatures that have been trusted by the Arch Dev team or specifically by you. The other option, `TrustAll` will accept any signature. `TrustAll` simply means "As long as the package is signed, I don't care by whom." The signature does need to be valid, but it doesn't need to be trusted. This is a rather long and complicated subject, and I'd rather not touch on it here. Suffice it to say you should never use `TrustAll` on the official repos, and I would recommend not using it on a local repo either, preferring no signature checking to `TrustAll` myself, but that's a decision you can make for yourself. I will say that for a personal repo set up within your own network, I suspect the attack vector to be so small as to be essentially zero, not to mention the risk is low. If you did want to take a run at setting up package signing, I would suggest starting with unsigned databases and packages anyways to make sure you understand the principles first, and add signing later on.

For the purposes of this guide, we will not be using database or package signing on our custom repos. There are references in the bibliography that you can read if you'd like to set them up.

### Custom Local Repos
We now know everything we need to know about making our own repos. We know we need a directory for our repo, we know we need a database in an archive format (`.db.tar` and optionally compressed) stored in that directory, and we know we need our package files stored there as well. We also know we need to define a new repo in `pacman.conf` by putting it in square brackets and then pass our settings in the lines after it, which at a minimum must be a path to our repo. As mentioned, we will also configure thee repo to not require database or package signing.

#### Local Repo Creation
So the first thing we will do is create a directory to hold our empty repo. You can put this anywhere you want. I will put this repo beside where pacman stores the official repos by default (`/var/cache/pacman/pkg` is this default location). I'll also, for simplicity, call my repo `localrepo`. We also want to ensure we have ownership of this directory and the repo or we'll have to use sudo every time we want to add things to it, which is not ideal if you use my `aurutils` method to maintain repos as described in the [Recommendations](#recommendations) section.
```bash
% sudo install -d /var/cache/pacman/localrepo -o $USER
% repo-add /var/cache/pacman/localrepo/localrepo.db.tar
```
That's it. That's all it takes to create a repo. I know I've written a lot already, but it really is that simple. 2 commands and you are the proud owner of your own repo. It may be empty and pacman might not know anything about it, but that'll change soon enough.

#### Local Repo Configuration
Now we need to tell pacman how to find our repo and what settings we want to use. This is very simple: we define a new repo block in `/etc/pacman.conf`, we set up our package signing options, and then we give it a link to our directory containing our database and, eventually, all our package files. The server location requires a protocol too. We're used to seeing `http://` or `https://`, but there are many options, like `ftp://`. In this case, it's just a local file, so we'll use `file://` followed by the full path to our directory containing our repo. So lets append the following lines to the end of `/etc/pacman.conf`:
```bash
[localrepo]
SigLevel = Never
Server = file:///var/cache/pacman/localrepo
```
And we're done. A quick `pacman -Syu` to update our databases (remember `pacman -Sy` is all we need to do to sync databases, but then we run the risk of forgetting we did it and accidentally doing a partial update, so I always use `Syu` to update my databases) and we're done. You should have noticed your empty `localrepo` database was synced as well.

#### Local Repo Usage
With a local repo created and pacman able to find it, the next step is to learn how to use our local repo. There aren't a lot of operations we need to worry about. For our purposes, we will want to be able to add packages to it, upgrade packages in it, and remove packages. To add or upgrade packages is actually the same command. I built the `aurutils` package from the AUR snapshot earlier, so I'm going to use that in my examples. First, let's add it to my repo. To do that let's copy it to our repo's directory, then use the repo-add command (which we also used to create an empty repo) to add it to our repo's database.
```bash
% cp ~/aurutils/aurutils-*.pkg.tar.gz /var/cache/pacman/localrepo
% repo-add /var/cache/pacman/localrepo/localrepo.db.tar /var/cache/pacman/localrepo/aurutils-*.pkg.tar.gz
```
I wouldn't recommend using the * wildcard like I did in the `aurutils` package filename, but since your version numbers may differ, this was more generic. In any case, we're done. Again, really simple to do (though it gets easier - see my discussion on aurutils in the [Recommendations](#recommendations) section) but extremely powerful. How powerful? If we ran `pacman -Syu aurutils`, pacman will install it. And this is the key to using a local repo.... pacman can only install packages, and the AUR doesn't host packages only snapshots, but because we built our package and stuck it in our own repo, pacman is happy to install it just as it would any of our core packages. That means even if we had a package that depended on `aurutils`, we wouldn't get an error from pacman if we tried to install it, because pacman can find `aurutils` in our local repo now.

If we had an older version of aurutils in our local repo already, this same repo-add command would update it. At which point I delete the old version - I personally don't bother keeping old versions of most packages in my repos. Any old version you did install would still be in your pacman cache anyway.

To remove a package from our repo, we just use repo-remove.
```bash
% repo-remove /var/cache/pacman/localrepo/localrepo.db.tar aurutils
```
Again, super simple. Note you don't pass the version number pr a package filename when you're removing a package from a repo.

You might be thinking: "Yeah, this isn't super long and complicated, but it's so much more than just typing `yay -S aurutils` so why bother?" I won't answer that right now, but I will again refer you to the [Recommendations](#recommendations) section to see my discussion on aurutils and realize how much more powerful and seamless this method is, especially when combined with what I call a "hosted repo" across your network. Let's discuss that next.

### Custom Hosted Repos
A custom hosted repo is actually extremely simple. So simple, in fact, that it only requires setting up a lightweight webserver with default options and toying around with permissions, and you get all the power of being able to serve your own packages (including your own built AUR packages) across your network or even over the internet.

#### Webserver Installation
As previously mentioned, there are a lot of ways to serve your repo across your network. For me the quickest and easiest is to set up a very simple and lightweight webserver. I use lighttpd for this and I've found I need zero configuration for it to work. Let's install it and configure it:
```bash
% pacman -S lighttpd
% systemctl enable --now lighttpd
```
Doesn't get any easier than that. Even file-sharing would take longer to configure. To check it's working, I create a simple page with `echo stoicaviator > /srv/http/index.html` and then fire up a browser and go to `127.0.0.1` and am greeted by "stoicaviator". I usually make sure I can access that page from other computers on my network using either the computer name or ip address. I personally have a DNS server, and I've set it up so that `http://repo.stoicaviator.com` points here. You can always just stick something in your hosts file to make this work, but this is basic networking so I'll leave that to you to figure out. In any case, once you are able to access your newly-minted `index.html` from your network, we can move on to making this work with a repo.

#### Hosted Repo Creation
Much like how we created our local repo, we need to start by creating a directory. In this case though, I'm going to put the directory under the webroot, more specifically at `/srv/http/stoicaviator` and I'm going to name my repo `stoicaviator`. Now is where thins get a little complicated because permissions become a bit of an issue. The problem here is that we want our user to be able to add things to the repo, but we also want the webserver to be able to access the database and the package files in the repo to serve them out. I find this works best by making my user the owner of the directory and files, with the webserver having group ownership and read permissions. I also use the setgid bit so the group is always set automatically.
```bash
% sudo install -dm2750 /srv/http/stoicaviator -o $USER -g http
% repo-add /srv/http/stoicaviator/stoicaviator.db.tar
```
Hosted repo created. Onto configuration.

#### Hosted Repo Configuration
In the case of a hosted repo, we need to configure it differently depending on the situation. On the machine that's hosting it - the one with the webserver installed - we need to configure it exactly as we did with a local repo because for that machine, it is local. For our client computers though, we configure it as a remote repo. The difference is only one line though, so it's not very difficult. That means on the host computer we need to add this to our `/etc/pacman.conf` file:
```bash
[stoicaviator]
SigLevel = Never
Server = file:///srv/http/stoicaviator
```
For our client computers, we need to specify the `Server =` as URL. You can do this by IP and computer name or however you set up your network so you could access the `index.html` page we created in the webserver setup section. As I said, for me, that was `http://repo.stoicaviator.com`. That means I need to add the following to my `/etc/pacman.conf` files on my client computers:
```bash
[stoicaviator]
SigLevel = Never
Server = file://repo.stoicaviator.com/stoicaviator
```
Note we also changed the repo name to match with our database name. On both the host and the clients now, you should be able  too run `pacman -Syu` and have the empty `stoicaviator` database sync across.

#### Hosted Repo Usage
So this is the pinnacle. This is where you see why you really should be using a hosted repo for your packages. The usage here is identical to with our local repos, because we only manage our repo from the host, and because as far as the host is concerned, it is a local repo. So let's install aurutils to our hosted repo now.
```bash
% cp ~/aurutils/aurutils-*.pkg.tar.gz /srv/http/stoicaviator
% repo-add /srv/http/stoicaviator/stoicaviator.db.tar /srv/http/stoicaviator/aurutils-*.pkg.tar.gz
```
Now head over to a client computer, run `pacman -Syu` to sync the databases. If everything worked, your client now treats `aurutils`, a package we previously had to build and install manually from the AUR or using an AUR helper, as if it was any other package. You can test that with either a search `pacman -Ss aurutils` or by installing it with `pacman -Syu aurutils`.

With all this configured, I no longer use an AUR helper on my client machines. Whenever I want an application that's on the AUR, I ssh to my host and make the package there, adding it to my repo, than I just use `pacman -Syu packagename` back on my client to install it. I'll remind you as well to check my [Recommendations](#recommendations) section where I discuss `aurutils`, and you can see how with 1 command I downloaded AUR snapshots, make packages from them and install them in my repo. Also with 1 command I update all packages in my hosted repo. And that means all my client computers ever need to do is `pacman -Syu` and that updates all their packages - official and AUR alike (as well as my custom packages).

With a hosted repo under our belt, we have just made meta-packages the logical next step. Now we can combine official and AUR packages into one big meta-package to easily duplicate our installed packages on new computers or fresh reinstalls, and they all work seamlessly thanks to our hosted repo.

## Meta-Packages
As we've already discussed, a meta-package is a package that contains no applications or files itself, but simply links multiple other packages together via dependencies. It's a way to install a set of applications all together and administered by a single package. 

Let's look at, for example, `plasma-meta`, which pulls in a minimum set of packages for a functioning KDE plasma environment. While it installs nothing itself, it has 28 dependencies, meaning in order to replicate what `plasma-meta` does, you'd have to install 28 other packages individually. That makes for a pretty long command to have to type in. Obviously remembering this big list is nearly impossible, so you'd probably create a text file of packages and pipe that to pacman instead, but now you have a file you have to remember to copy over to new installs to use. Not only that, but now if you ever go through your list of explicitly installed packages looking to slim down your system, or looking for something specific, you've got 28 packages in the place of one. A meta-package streamlines all this.

It's worth noting as well that a meta-package can pull in any package your pacman knows about. On a fresh, default install, that means your meta-package can only contain packages from the official repos. But with a hosted repo, like we've set up, all you need to do is ensure that `/etc/pacman.conf` is set up with our hosted repo and we have access to any package we've also put in there - which may be packages from the AUR, or even packages we've created ourselves, including meta-packages. That last point, "including meta-packages", is actually a huge advantage. Imagine we create a PKGBUILD for a package called `stoicaviator-meta`, and we have it set up to pull in a set of applications we regularly use and want installed on all of our systems, we can build `stoicaviator-meta` using ABS and stick it on our hosted repo, and now we can even use it to install a base system. Let's take a loot at how that would work.

### Meta-Package Template
The only file you need for a meta-package is a PKGBUILD file, of which only a basic amount of information is required. Here is the template I personally use for a meta-package:
```
# Maintainer: Stoic Aviator <stoicaviator[at]gmail.com>
pkgname=
pkgver=
pkgrel=
pkgdesc=
url='https://github.com/stoicaviator'
arch=('any')
license=('GPL')
depends=()
```
As you can see, it's very simple. We only need to fill in 5 lines to create a meta-package. Let's take a look at those fields we need to complete.

`pkgname` is a simple name that the final package will be called - this would be what you pass to pacman to install it.

`pkgver` is almost always just a number, usually in decimal notation, indicating the version, though it can contain text as well. I'll just recommend you don't use anything other than numbers and decimals here, as doing so requires you write an additional function letting pacman know when a package has an update - is `myversion` newer than `anotherversion`? I start with 1 usually, some people start with 0.0.1, other with 1.0.0. It's up to you. We'll see an example shortly. This is one of the two ways that pacman (or an AUR helper in the case of package snapshots on the AUR) will know an application has been updated. A bigger number means a newer package, so always increment this when you make a change. Going from, for example, 2.3 to 1.1 will not cause an update. But going from 1.0.0 to 1.0.1 or from 2 to 3.1 will. In general the bigger the jump, the more significant the change. I tend to stick with integers and use this field to indicate only major changes. So small changes I indicate  with `pkgrel`, which we look at next, but a major change would cause me to bump my `pkgver`.

`pkgrel` is the release of the package. In the case of official packages, this usually implies a correction. For example, if we were an Arch Maintainer and we made a package for the Linux Kernel, we wouldn't want to change the version to fix an error with our package or because we adjusted an Arch-specific patch, because then there would be confusion about which upstream kernel we are referring to. If we had a package to install the Linux Kernel, version 5.4.1, but wanted to change a config option and changed our `pkgver` to 5.4.2, what would we do when upstream release the 5.4.2 version of the Linux Kernel? So instead, we use the "release number". I use this to indicate small changes to my package, as we'll see shortly.

`pkgdesc` is just a text field that describes what this package does.

`depends` is where the magic happens - this is where you list all the packages your meta-package will contain. They are all installed as dependencies as the name implies. As previously mentioned, if you include packages here that your installation can't see, such as an AUR package that isn't installed in a hosted or local repo, you'll get errors trying to install it as unfulfilled dependencies. So again, this is why a self-hosted repo really is the way to go if you want to be able to put anything you want here.

### Create A Custom Install Meta-Package
One of the most common ways people use meta-packages is to create a base install with all their applications of choice. This allows you to replicate a desired minimal install without worrying about forgetting something. That's the next step. We know that the `base` meta-package is missing a few things: a kernel, the firmware, filesystem utilities, a text editor, etc. Let's make a meta-package to remedy that situation. Let's set up our project on our computer with our self-hosted repository by creating a folder for it and the PKGBUILD.
```bash
% mkdir ~/stoicaviator-base
% cd ~/stoicaviator-base
% touch PKGBUILD
```
The first thing I want to do is include `base`, because otherwise I have an unsupported install. I'll include the vanilla kernel (`linux`), the firmware (`linux-firmware`), utilities for the filesystems I use (`dosfstools` for VFAT, `e2fsprogs` for ext2/3/4, and `cifsutils` for samba shares), my text editor of choice (`vim`), and the man page utilities and data (`man-db`, `man-pages`, and `texinfo`). Since I use `systemd-networkd` and `systemd-boot` for my network and bootloader in the vast majority of cases, I don't need to include any networking utilities or a bootloader, but you could definitely add them as well. So here's the content of our complete `PKGBUILD` file:
```
# Maintainer: Stoic Aviator <stoicaviator[at]gmail.com>
pkgname=stoicaviator-base
pkgver=1
pkgrel=1
pkgdesc='Minimal set of packages required for a Stoic Aviator base install.'
url='https://github.com/stoicaviator'
arch=('any')
license=('GPL')
depends=('base' 'linux' 'linux-firmware' 'dosfstools' 'e2fsprogs' 'cifsutils' 'vim' 'man-db' 'man-pages' 'texinfo')
```
Now we can build our package, and stick it in our hosted repository.
```bash
% makepkg -s
% cp stoicaviator-base-1-1-any.pkg.tar.xz /srv/http/stoicaviator
% repo-add /srv/http/stoicaviator/stoicaviator.db.tar /srv/http/stoicaviator/stoicaviator-base-1-1-any.pkg.tar.xz
```

### Use Our Custom Meta-Package
We now have our meta-package available to any computer with access to our repo. So if we wanted to do a fresh install on a different computer, we would follow the Wiki install until we get to "Select Your Mirrors". After we've done that in our `/etc/pacman.d/mirrorselect` we just need to add our hosted repo to `/etc/pacman.conf` file:
```
[stoicaviator]
SigLevel = Never
Server = http://repo.stoicaviator.com/stoicaviator
```
Instead of pacstrapping `base`, and a kernel, and a text editor, etc, we can just pacstrap out custom meta-package and get everything in one:
```bash
# pacstrap /mnt stoicaviator-base
```
Once we continue with the setup, we already have all of our tools installed. This may not seem very exciting, but remember you can build packages from the AUR and put them in your custom repo, and then they are available in a meta-package, and suddenly this becomes a lot more powerful. Let's go ahead and update our meta-package to demonstrate this.

### Modify our Custom Meta-Package
Let's say we want to make a small change to our meta-package. Maybe I've decided I prefer `zsh` as a shell (which I do) and want to add it to my base package. How would I do that? I just add it in to the array, bump my `pkgrel` since it's a minor change, then build my package again and re-add it to my hosted repo to update it. So my new PKGBUILD looks like this:
```
# Maintainer: Stoic Aviator <stoicaviator[at]gmail.com>
pkgname=stoicaviator-base
pkgver=1
pkgrel=2
pkgdesc='Minimal set of packages required for a Stoic Aviator base install.'
url='https://github.com/stoicaviator'
arch=('any')
license=('GPL')
depends=('base' 'linux' 'linux-firmware' 'dosfstools' 'e2fsprogs' 'cifsutils' 'vim' 'man-db' 'man-pages' 'texinfo' 'zsh')
```
So let's build and add it to our repo:
```bash
% makepkg -s
% cp stoicaviator-base-1-2-any.pkg.tar.xz /srv/http/stoicaviator
% repo-add /srv/http/stoicaviator/stoicaviator.db.tar /srv/http/stoicaviator/stoicaviator-base-1-2-any.pkg.tar.xz
```
You'll notice repo-add tells you that your package has been updated to the new version. Now go to any computer we used this on, that has `stoicaviator-base` installed on it, and run `pacman -Syu`, and you'll notice that `zsh` gets installed and `stoicaviator-base` is updated from 1-1, to 1-2. With a simple system update, your meta-package has also updated.

### Integrate AUR into Custom Meta-Package
There are few things from the AUR that I always like to have installed. A few fonts: `nerd-fonts-complete`, `ttf-iosevka` and `ttf-ms-fonts`. A different kernel and the headers: `linux-ck`, `linux-ck-headers`. A few apps: `spotify-tui` and `6cord`. Maybe a game too: `2048-rs`.

The first step is to add this to our hosted repo. If you still haven't checked the [Recommendations](#recommendations) section to see what I have to say about `aurutils`, perhaps showing you how powerful it is now to add all of these packages to our repo with practically no effort will convince you to go check it out. First we install `aurutils` since we built it earlier but never installed it:
```bash
% pacman -S aurutils
```
Now we can just use `aur sync` to install our packages. The one snag here is that we have two local repos defined currently on our host. I would personally remove the `localrepo` and keep only the reference to our hosted repo `stoicaviator`. You could specify which database to use by passing `-d stoicaviator` to `aur sync`, but I'm just going to assume you've removed `localrepo` since there is no point having 2 of them. I'm also going to build in a clean chroot, which you've probably heard being mentioned, by passing the `-c` option. I also don't want to keep build and make dependencies installed by `makepkg` when `aurutils` calls it, so I'll also pass the `-r` option. This is how it works:
```bash
% aur sync -rc nerd-fonts-complete ttf-iosevka ttf-ms-fonts linux-ck linux-ck-headers spotify-tui 6cord 2048-rs
```
All your packages were built, in a clean chroot even, copied to your repo's directory and added to your repo's database. The only problem with this method is that your package files are owned by you and your group, not you and the http group, which means your server won't be able to send them to clients that want them as it won't have permission. If you're read my recommendations, you know how to fix this permanently so you never have to worry about it. For now, let's just change the ownership.
```bash
% sudo chown $USER:http /srv/http/stoicaviator/*.pkg.tar.gz
```
It's also worth noting that instead of worrying about updates to your packages from the AUR, you can also just run `aur sync -rcu` to update all the AUR packages in your custom repo too, and that means when you `pacman -Syu` on your clients, they will all download updated AUR packages too. It cannot be easier to maintain. 

Now we can edit our meta-package. Since this is a pretty big change, let's go ahead and bump our package version with this change (notice `pkgrel` is reset to 1 - we restart the numbering here with every new `pkgver`). We also can update our dependency array (I'm also removing `linux` since I'm going to be using `linux-ck` instead). You'll also notice I've broken up my depends array to make it more readable now:
```
# Maintainer: Stoic Aviator <stoicaviator[at]gmail.com>
pkgname=stoicaviator-base
pkgver=2
pkgrel=1
pkgdesc='Minimal set of packages required for a Stoic Aviator base install.'
url='https://github.com/stoicaviator'
arch=('any')
license=('GPL')
depends=(
    ## Official Packages ##
    'base' 'linux-firmware' 'dosfstools' 'e2fsprogs' 'cifsutils' 'vim' 'man-db' 'man-pages' 'texinfo' 'zsh'
    
    ## AUR Packages ##
    
    # Fonts
    'nerd-fonts-complete' 'ttf-iosevka' 'ttf-ms-fonts'
    
    # Kernel-related
    'linux-ck' 'linux-ck-headers'
    
    # Apps
    'spotify-tui' '6cord'
    
    # Games
    '2048-rs'
    )
```
Now we build it and add it to our repo:
```bash
% makepkg -s
% cp stoicaviator-base-2-1-any.pkg.tar.xz /srv/http/stoicaviator
% repo-add /srv/http/stoicaviator/stoicaviator.db.tar /srv/http/stoicaviator/stoicaviator-base-2-1-any.pkg.tar.xz
```
Again, a simple `pacman -Syu` on all our clients, and they pull in everything, including our packages we built from the AUR. And this still works with a fresh install because, as I've attempted to make clear, the only reason pacman can't install "AUR packages" is because the AUR doesn't actually hold packages, but now that we've turned the ones we want into packages and put those package files in a database that pacman has access to, it treats them like any other package.


## Recommendations
I have a small list of recommendations. These are just things I have found to be the most useful for me. I am by no means an authority on any subject related to Arch, so take them all with a grain of salt.

### Use Aurutils
The number one piece of advice I have is this: use `aurutils`. In truth, any AUR helper that uses a local repo would work, but of those `aurutils` has been around the longest and is the most well tested, and it's creator/maintainer, AladW, is a Trusted User by the Arch Dev Team. I would suggest building `aurutils` once, ideally in a clean chroot, and hosting it on your hosted repo. I personally wouldn't add it to my meta-packages because I only host packages from the AUR on my server with the hosted repo, even if they are only used on one machine. I just find it more convenient to keep it all in one place. I regularly go through my host repo and remove any packages from the repo database and delete the files that I no longer need, so having it all in one place helps.

Perhaps the best reason to use AUR utils isn't even that you can automatically add packages from the AUR to your hosted repo upon building them with one command, but that it lets you keep all your AUR packages updated with a simple command. I'll leave a deep dive into the working of `aurutils` to the user, but suffice it to say that if you're using a hosted repo, you won't find an easier way to maintain things.

The only snag is that packages added to your repo with `aurutils` will have the ownership set to $USER:USER, but we want the ownership to be $USER:http. There are several ways to tackle this problem, from setting lighttpd's user to $USER, to using FTP or another protocol that is less picky, to doing some strange mounting things... I for one just put a function in my `.bash` and `.zshrc` files that takes care of it for me. It looks like this:
```bash
function aursync {
    aur sync -rc "$@"
    sudo chown -R $USER:http /srv/http/stoicaviator/*.pkg.tar.xz
}
```
This is not an elegant solution, but it works for me. So now instead of calling `aur sync`, I just call `aursync` and everything works. To install a new package from the aur, `aursync yay`, and to update all my packages in my hosted repo, `aursync -u`.

### Host AUR Packages
I no longer install AUR helpers and build packages from the AUR on my client machines if at all possible. I have `aurutils` on my computer which serves my hosted repo out, and when I need an AUR package, I build it there and serve it out via the repo. This helps me keep my client machines clean and gives me a central place to manage my packages. It also allows me to have those packages available to any other computer that may want them. "Build once" has basically become my motto - instead of building a package off the AUR multiple times on different machines, I build ti once and then they all get the update from the server. I find this really simplifies my workflow and helps me keep track of what I've been doing.

### Use a Version Control System
If you aren't familiar with VCS, pick one and learn it. I use mainly git and host on GitHub, but there are plenty of options for VCS software and for hosting locations. Hosting your applications on VCS makes it easy to try new things and to maintain central control of your applications, as well as the added security of knowing they are backed up to the cloud. Learning how to use a VCS will be immensely useful to you once you start to realize it's potential. For keeping copies of your PKGBUILDS it is invaluable.

### Consider Security
Just because we've configured a simple webserver to share packages that can easily be extended to share them over the internet, doesn't mean you should automatically do that. This was a basic overview of the power of meta-packages and does not mean that best-practices have been implemented. I personally have my `aurutils`, repos and pacman setup to use package-signing. Both my databases and my packages are all signed and that is enforced within my network and on the machines I control. While this may not be necessary for you, it's worth considering. I've also hardened my webserver but haven't covered that here. Make sure you do your research and secure your computers and resources to the degree you feel is required.

## Conclusions
This may seem like we've done a lot of work, we have reduced our workload on adding new packages to a PKGBUILD edit, building and adding our meta-package to our repo, and maybe a single `aur sync` command away if it's a package from the AUR we want to add, from updating and maintaining our systems. This is far easier than any other way to build systems with a common base than any other, including git submodules or text files being parsed into `yay`.

At this point, you should understand the difference between a package and a package snapshot, between groups, packages and meta-packages, how pacman works, how the AUR works, and how to create and maintain your own repositories. There's really no limit to what you can do with this. I, for example, have several inter-woven packages. I have a `stoicaviator-base` meta-package which includes all my base utilities and I pacstrap to new installs instead of `base`. I have a `stoicaviator-xorg` which depends on `stoicaviator-base` and pulls in everything needed to get X up and running. I've also got `stoicaviator-amd` and `stoicaviator-nvidia`, both of which include the preferred drivers and utilities for either AMD-based GPUs or nVidia-based GPUs. I use the `provides` functionality of the ABS in my PKGBUILD files for those two meta-packages to create a dependency for `stoicaviator-gpu` which either one will satisfy. I then have `stoicaviator-bspwm` and `stoicaviator-plasma`, both of which depend on `stoicaviator-gpu` and provide everything I want for my bspwm or KDE plasma installs respectively.

So my final demonstration of the power of meta-packages, is to show you my pacstrap command I use during an install to install a KDE plasma environment:
```base
# pacstrap -i /mnt stoicaviator-plasma
```
That's it (the `-i` makes it interactive so I can pick which `stoicaviator-gpu` provider I want, among others). And so when I'm finished with my install and I reboot, I can get a list of all explicitly installed packages:
```base
% pacman -Qqte
stoicaviator-plasma
```
That's it. I have a working install, with KDE, games (steam and DF), mpd, drivers, libraries, programming languages and interpreters, a web browser, a media-player, OBS, discord, text editor, file utilities, shell, kernel, etc all installed with a single app. And any changes I want to make I do by updating that package in my repo and just running `pacman -Syu` on my client. The flexibility and power of this method is without equal. Combined with a dot-file manager, and I need only backup a few select directories in my $HOME folder.

I hope this has been useful and informative, and I hope you now have the confidence to host your own repo and maintain your own meta-packages.

## Bibliography
- [Arch Build System](https://wiki.archlinux.org/index.php/Arch_Build_System)
- [Arch Package Guidelines](https://wiki.archlinux.org/index.php/Arch_package_guidelines)
- [Arch User Repository](https://wiki.archlinux.org/index.php/Arch_User_Repository)
- [AUR Helpers](https://wiki.archlinux.org/index.php/AUR_helpers)
- [Building in a Clean Chroot](https://wiki.archlinux.org/index.php/DeveloperWiki:Building_in_a_clean_chroot)
- [Creating Packages](https://wiki.archlinux.org/index.php/Creating_packages)
- [General Recommendations](https://wiki.archlinux.org/index.php/General_recommendations)
- [Git](https://wiki.archlinux.org/index.php/Git)
- [Frequently Asked Questions](https://wiki.archlinux.org/index.php/Frequently_asked_questions)
- [makepkg](https://wiki.archlinux.org/index.php/Makepkg)
- [Pacman](https://wiki.archlinux.org/index.php/Pacman)
- [Pacman/Package Signing](https://wiki.archlinux.org/index.php/Pacman/Package_signing)
- [Pacman Tips and Tricks](https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks)
- [PKGBUILD](https://wiki.archlinux.org/index.php/PKGBUILD)
- [System Maintenance](https://wiki.archlinux.org/index.php/System_maintenance)
