## Coming Up With Project Ideas (Naturally)

Let's face it, coming up with ideas to build something new and useful is hard. It becomes even harder when an idea starts to seem a bit promising and someone has already made it. So now-a-days, I've stopped looking for ideas and taken a completely different route.

In this article, I'll discuss **one** very simple yet effective way of coming up with new ideas for your next side-project. That is —

> Instead of looking for new ideas look for unsolved problems. Figure out a solution and keep that as a project.

As software developers, we're translators between real-life issues and technical solutions. So the above mentioned technique seems like the most natural way to me.

Now the question is how do you track down these issues. The answer is you don't, they'll present them to you. These problems I'm talking about may range from something as small as *batch resizing images in a directory* to *building a system that allows you to run compiled binaries on any platform.*

Now you may think *if a problem is so prevalent, why haven't someone already attempted to fix it?* or *if the solution can be as simple as one of my side-projects is it really worth it?* and to answer these questions, you have to patiently stick with me till the end of this article.

I've come equipped with few case studies to proof my point. These case studies are open-source projects that I've developed solo or along with one of my partners. I'll present the problems I faced and how I came to develop these solutions. Without further ado, let's being with the first one —

### [fhsinchy/tent](https://github.com/fhsinchy/tent)
Tent is a open-source development-only dependency manager for Linux. What it does is it lets you run regular development dependencies like a [MySQL](https://www.mysql.com/) or [MongoDB](https://www.mongodb.com/) server with a simple one liners —

```bash
tent start mysql

# Which tag do you want to use? (default: latest): 5.7
# Server Port? (default: 3306): 3308
# Server Root Password? (default: secret): password
# Server Data Volume? (default: mysql-data): mysql-data-57
# Creating tent-mysql-5.7-3308 container using docker.io/mysql:5.7 image...
# INFO[0013] Going to start container "e1031dbfcbc11be6f52524dccd41c7e01057661a9ddb3f4a26b127d79143dd64"
```

Now you have a MySQL 5.7 server running on your computer that can be accessed on port `3308` using any client.

![Using MySQL Workbench](https://cdn.hashnode.com/res/hashnode/image/upload/v1618069675251/boLgLPlRG.png)

It's even easier if you go with the default options. To to spin up a MongoDB server using the [latest](https://hub.docker.com/_/mongo?tab=tags&page=1&ordering=last_updated) image available on [Docker Hub](https://hub.docker.com/_/mongo), you can execute the following command —

```bash
tent start mongo --default

# Creating tent-mongo-latest-27017 container using docker.io/mongo:latest image...
# INFO[0000] Going to start container "f2379a48400d9318c806a920f1029d9be00468485702956328527d954c4ab9b5"
```

And now I have MongoDB running on my computer accessible on port `27017` using any client.

![Using MongoDB Compass](https://cdn.hashnode.com/res/hashnode/image/upload/v1618070247744/zbdxGwO6jg.png)

Shutting these servers is even easier  —

```bash
tent stop --all

# Stopping e1031dbfcbc11be6f52524dccd41c7e01057661a9ddb3f4a26b127d79143dd64 container...
# Removing e1031dbfcbc11be6f52524dccd41c7e01057661a9ddb3f4a26b127d79143dd64 container...
# Stopping f2379a48400d9318c806a920f1029d9be00468485702956328527d954c4ab9b5 container...
# Removing f2379a48400d9318c806a920f1029d9be00468485702956328527d954c4ab9b5 container...
```

If you want to learn more about this program you can read this  [article](https://podman.io/blogs/2021/02/08/easy-development-dependency-management-with-podman-and-tent.html) on the official Podman [blog](https://podman.io/blogs/).

To be honest, Tent is not an original idea. In fact it's inspired directly from another tool called  [Takeout](https://github.com/tighten/takeout) by [Tighten Co.](https://tighten.co) which is written in PHP and uses Docker as its back-end.

I came across Takeout when I was done with installing dependencies on my different machines often running different operating systems. Initially I was using custom Dockerfile(s) to spin up these dependencies and it was cumbersome to say the least. Takeout which to be honest is an amazing program, solved a lot of these issues and was working flawlessly.

Until on December, 2020 I decided to migrate to [Podman](https://podman.io) from  [Docker](https://docker.com) as my go-to containerization tool and was looking for an alternative to Takeout that works with Podman. Another reason was that many of the developers in our team are not PHP developers at all. They simply don't want to set-up PHP, Composer and all the necessary dependencies on their machines. Some of them were using Vagrant and that was no better solution than Takeout.

So I started thinking about writing a program that serves as an alternative to Takeout, uses Podman as it's back-end and comes as a compiled binary. While researching I stumbled upon the [official golang bindings](https://pkg.go.dev/github.com/containers/podman/v3) for Podman and immediately knew what I had to do.

It took me about two weeks to bring Tent in a stable state and I learned so much about [Go Programming Language](https://golang.org) along the way. Since it's first release, Tent has become one of the most essential tool not only in my personal workflow but also in my office.

My colleagues have already migrated to it from whatever they were using before and have been pretty satisfied with it so far. Thanks to this project I've also got the chance to join one of the official community meetings and showcase my little program. The [recording](https://bluejeans.com/playback/s/m9virlPgDNUpxpzw8PTHCAF3rx4JAxx2bgEulenNJb88AlmTK5c3fSAyEfLp0DMP) is available on BlueJeans Network. In currently has more than 70 stars on GitHub and 30 or so downloads. For a niche project like Tent and given Podman has not been as widely adopted as Docker, I'm happy with it's state. 

## [fhsinchy/mcpkgrm](https://github.com/fhsinchy/mcpkgrm)
This program is a bit personal. Back in 2020 I used to own a MacBook Air and one of the annoyances I used to face was the uninstallation process of applications installed using `.pkg` packages.

For those of you who don't know, MacOS applications often come bundled in `.pkg` packages which acts a lot like `.msi` packages on Windows. Now the problem with these packages is that they can not be uninstalled very easily unless the installer comes with an uninstallation script of it's own.

During my usage, I only found two ways to uninstall such a package. I could either execute a series of complex bash commands or I could buy [UninstallPKG](https://www.corecode.io/uninstallpkg/) application by spending $9.99.

The fact that I have to pay money just to uninstall a freakin' application was annoying enough for me to rise up and write an uninstaller of my own. Hence, `macOS(mc) Package(pkg) Remove(rm)` was born.

It's a simple Python program that presents you with a list of all the installed `.pkg` in your system and lets to uninstall them **for free** —

![List of Installed Applications](https://cdn.hashnode.com/res/hashnode/image/upload/v1618072110095/C1-y7drtF.png)

Although I sold my MacBook soon after and development of this program came to an end, I've received a few messages on [Reddit](https://reddit.com) thanking me for writing the program.

### [fhsinchy/rmbyext](https://github.com/fhsinchy/rmbyext)
This is a very small (40 lines long) Python script that lets you delete all files of a given extension recursively. The name `Remove(rm) by Extension(ext)` literally stands for that.

I mean we all have faced situations where we have to go inside directories and nested directories to hunt down files of a certain extension. I do this a lot and `rmbyext` has become one of the most used program on my computers. Although this was written specifically to fill my needs, I've got 9 stars on the repo maybe because 9 people have found it useful as well.

### [fhsinchy/opengapps-unofficial-builds](https://github.com/fhsinchy/opengapps-unofficial-builds)
This is one of my oldest projects. Prior to December 22nd, 2017, [The Open GApps Project](https://opengapps.org) didn't have support for Android 8.1 Oreo but many custom ROM providers including [LineageOS](https://lineageos.org) had already released their nightlies and betas to the world. This created a situation where people had necessary firmware but didn't have Google Applications to go with it.

One of the sufferers was me who had just installed LineageOS on my OnePlus X and found myself stranded. So I rolled up my sleeves, built GApps from source with support for API v27 and released them to all my fellow warriors.

The [XDA Thread](https://forum.xda-developers.com/t/gapps-8-1-opengapps-unofficial-builds-arm-arm64.3723022/) for this project has since received thousands of views and downloads. The project even crossed the line of One Plus X forum and started popping up on forums of other devices, filling up my notification panel in the process. Open GApps finally added support for API v27 on 22nd December, 2017 hence I set the sun for my project.

The last comment on the thread was on May 3, 2018 at 11:53 AM by a forum moderator and it went something like as follows —

![CamoGeko](https://cdn.hashnode.com/res/hashnode/image/upload/v1618073376867/4wn3iVMCX.png)

Well that made me feel like nothing less of a hero and I still get respected on the forums.

### [sayburgh-solutions/mongoose-permissions](https://github.com/sayburgh-solutions/mongoose-permissions)
Few days back at work, we needed a simple but effective way to implement Role Based Access Control (RBAC) in a [Node.js](https://nodejs.org/) project. I looked for one online but the ones I found were either overengineered or lacking necessary features.

So I along with my colleague [M H Hasib](https://mhhasib.com) started working on a project of our own inspired from the API of [laravel-permissions](https://spatie.be/docs/laravel-permission) package by [Spatie](https://spatie.be/) and the result was [mongoose-permissions](https://www.npmjs.com/package/mongoose-permissions) package.

Code example using the package is as follows —

```js
const user = await User.findById('5fd7586ab8069d56e77e170e');

// the assignRole() method takes a complete role object as it's input.
// assigning a new role automatically replaces the old one.
user.assignRole({
        name: "Editor",
        permissions: [
            {
                name: "create-article"
            },
            {
                name: "edit-article"
            }
        ]
    });

// the hasRole() method takes the role name as input.
// the method returns true if the user has the role, false otherwise.
if (user.hasRole("Editor")) {
    // necessary logic goes here.
};

// whereas the revokeRole() method takes the role name as an input.
// revoking a role leaves the selected user with no permissions at all.
user.revokeRole("Editor");

// the givePermissionTo() method takes a permission name it's input.
user.givePermissionTo("publish-article");

// the can() methods takes the permission name as input.
// the method returns true if the user has the permission, false otherwise.
if (user.can("edit-articles")) {
    // article editing logic goes here.
};

// the revokePermissionTo() method takes a permission name it's input as well.
user.revokePermissionTo("publish-article");
```

After finishing the project, we were really happy with the API and since then it's running on our production servers with zero issues so far.

The package gets 11 downloads per week on average. Although those are rookie numbers, we're happy about the fact that we've wrote something that solves not only our issue but maybe someone else's as well.

### Closing Thoughts
That was the fifth and last case study in today's article. In the end I would like to give you three advice regarding your side-projects —

1. Don't be afraid if someone has already made a similar project. If you think that your reincarnation solves an issue that the previous didn't, go ahead and make that. Case in point my Tent project. I could've just stick to Takeout and avoided the hassle of developing a new program. But Takeout doesn't come as a compiled binary and it also doesn't support Podman as a back-end. I solved those issues with my program and became a winner at the end.
2. Even if you think the necessity of your side-project will diminish in near future, don't be afraid. Case in point my Open GApps project. I knew that the official version will arrive very soon but I made my builds anyway and ended up helping hundreds of stranded users.
3. If you feel like your side project is too much targeted at your own needs, make it anyway and make it public. Case in point my `rmbyext` and `mcpkgrem` projects. Although I made them thinking about myself only, they helped some other people as well.

The goal here is to build a solution. A collection of source files that compiles into something useful and I believe that you can do that. So `BEGIN` already, will ya?