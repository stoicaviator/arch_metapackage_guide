# A guide to using meta-packages with Arch Linux.

## Introduction

This guide is intended to give Arch Linux users who are new to the idea of meta-packages and self-hosted respositories (repos) the basics of maintaining their own packages and repos. This topic is immensely useful for anyone that has more than one computer and would like to sync package changes (not configuration changes) across them. The information is wholly available in the Arch wiki, but does not currently exist compiled in a single page. A full bibliography of Arch Wiki articles that contain pertinent information or that are referenced in the guide will be listed below as well as linked throughout the document as they are referenced.

### Table of Contents
-[Basics](#basics)
  -[Q1. What is a package?](##q1-what-is-a-package)
  -[Q2. What is a PKGBUILD?](#q2-what-is-a-pkgbuild)
  -[Q3. What is the Arch Build System?](#q3-what-is-the-arch-build-system)
  -[Q4. What is a dependency? Also, what does it mean to say a package is installed 'explicitely'?(#q4-what-is-a-dependency-also-what-does-it-mean-to-say-a-package-is-installed-explicitely)
  -[Q5. What is a meta-package](#q5-what-is-a-meta-package)
  -[Q6. What is a group?](#q6-what-is-a-group)
 
## Basics

### Q1. What is a package?

A package is a single compressed archive that contains all the information and files neccessary for the Arch Linux **pac**age **man**ager to install a specific application. For example, the package for the default shell `bash` is a compressed archive containing all the information about `bash` as well as the `bash` executable, a host of configuration files and the manual pages. The idea is that any computer containing all the pre-requisites for a given application (dependencies, architecture, pacman, etc) could, with only the package compressed archive, install the complete application. This is true for a small, simple application with one executible file and no additional config files or data as well as for a complex application with hundreds of files like Visual Studio Code.

You can identify package files easily because they have the `pkg.tar.xz` extension. Its important to note that all packages in Arch Linux are binary packages, meaning no source is downloaded or compiled when you install a package. For those who are just learning, you may think this statement is incorrect because you've installed packages that used git source, but if you read on you will where that confusion comes from. All the files hosted on the official Arch Linux repos (core, community, etc) are packages.

### Q2. What is a PKGBUILD?

A PKGBUILD is a file used by developers to guide the Arch Build System (see Q3) on creating a package from sources. It contains information like what version the build package will be, which the build package's description is, what files will be included in the built package, etc. It also contains any instructions that may be required on where to install files, or what script to run before installing an application. If a PKGBUILD is placed in a directory with all the required source files (or links to the correct location of the source files), invoking the Arch Build System will result in a single package file, in the form of a compressed acrhive, ready for distribution and installation to other users or systems.

### Q3. What is the Arch Build System?

The Arch Build System (referred to as the ABS), is the framework by which the Arch Linux development tools create packages from simple, easy-to-read instructions. Some packages require hundreds of files to be included and dozens of libraries to be installed, others are straightforward. It is a detailed, robust, extremely adaptable system that takes a long of diligent study to master. We will only examine a small portion of the Arch Build System in this guide.

### Q4. What is a dependency? Also, what does it mean to say a package is installed 'explicitely'?

There are two ways a package can be installed: explicitely or as a dependency. A package is installed explicitely when you directly tell pacman to install it. For example, `pacman -S base`. `base` is being passed explicitely to pacman. `base` needs some other packages to work properly too though. For example, it needs `filesystem`, which is the package that contains all the files that are required for Arch Linux to work that aren't part of other packages. A good example of this is `/etc/fstab`; it's not installed by any other package or application and yet Arch Linux needs this file to run, so it is included in the `filesystem` package. This requirement for `base` that `filesystem` needs to be installed is called a dependency. `base` depends on `filesystem` to work, so `filesystem` is a dependency of `base`. 

You can imagine `base` depends on a lot of things. And some of those things depend on other things. Much like a family tree, you can imagine how this system of dependencies might spread out, and this entire family of dependencies is called a dependency tree. Just as `base` depends on `filesystem`, `filesystem` depends on a package called `iana-etc`. So when you install `base`, first pacman installs `iana-etc` so that it can install `filesystem`, then it installs `filesystem` so that it can install `base`, and then it installs `base`. Of course, it also has to install all of the other dependencies to install `base` and they may have their own dependencies and so on, but you get the gist. A single explicitely-installed package could result in a huge dependency tree being installed to support it.

When pacman installs a package, it stores some simple information about it in its own local database on your computer. It does this to keep track of what packages are there, which is why when you run a system update with `pacman -Syu` it knows which packages you have: it has them all listed in a database. It stores a bunch of information, but for the purpose of this guide we need to know it stores the package name, the package version, the package release number and whether it was installed explicitely or as a dependency.

The reason to differentiate between explicit or as a dependency is because dependencies change. Imagine package `foo` depends on package `bar`. But a few years down the road something much better than `bar` comes out called `baz`, and `foo` stops needing `bar` but needs `baz` instead... we would now call `bar` an orphan - it was installed as a dependency to a package that no longer depends on it. Unless someone has made an error someone, orphas can be safely removed from your system, and many Arch users check for orphans on a regular basis.

### Q5. What is a meta-package

A meta-package is a term for a specific type of package that actually contains no applications or files itself, but rather serves as an umbrella that emcompasses a number of packages underneath it. In essence, a meta-package is a package that is completely empty except that it depends on other packages. This simplifies the installation (or removal) of multiple packages by installing them all automatically when the meta-package is installed. It also serves to make the list of explicitely installed packages smaller. For example, if we had a metapackage called `foobarbaz` consisting of the packages `foo`, `bar` and `baz`, we could install them all simply with `pacman -S foobarbaz`. If we output a list of explicitely installed packages, we would see `foobarbaz` in the list, but not `foo`, `bar`, or `baz`. We could see them all if we output a list of all installed packages, or packages installed as a dependency. We could also uninstall them all with `pacman -R foobarbaz`.

There is an important distinction here. Remember I said a package was a single compressed archive that contains all the information and files necessary for pacman to install a specific application? In this case, we have no files. The package `foobarbaz` **does not contain** the files for `foo`, `bar` or `baz`, only that they are dependencies. So if you took your `foobarbaz` package file to a computer with no internet access and tried to install it, pacman would complain about missing dependencies `foo`, `bar` and `baz` unless you had already installed them before. The `foobarbaz` package tells pacman it needs `foo`, `bar` and `baz`, but it does not contain their files.

Some users have complained about this. But if you think about it for a second, it only makes sense. Let's just show how unrealistic it is to expect packages to have their dependencies bundled in. Let's imagine you want to install the open-source version of Visual Studios. This package is called `code`. `code` depends on 13 other packages, most of which have their own dependencies. Several of them depend on `libx11`. `libx11` has 4 depenedencies, some of which are shared with dependencies of these 13 other packages too. And if you drill down, you would find dozens of these packages depend on `filesystem`, which as we know depends on `iana-etc`. That means you'd have dozens of copies of `iana-etc` bundled inside packages, which are bundled inside other packages, and bundled inside yet more packages, which are bundled in with `code`. So now every single package you try to install requires hundreds of MiB of downloads. That's not just time you spend downloading (and disk space), but internet bandwidth for you, internet bandwidth for the Arch servers, load on the Arch servers, etc. It is unreasonable to expect or demand that packages have their dependencies bundled in with them.

### Q6. What is a group?

A package group is like a meta-package, except that all members of the group are installed explicitely. Remember a meta-package only has dependencies, and while the meta-package and its dependencies all get installed by pacman, all the meta-package is installed explicitely. With a group, you don't end up with a package named after the group installed explicitely and a bunch of installed dependencies, you end up with all members of the group installed explicitely.

