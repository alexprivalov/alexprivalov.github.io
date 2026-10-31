---
layout: post
title: "Using Docker for Cross-Compiling C/C++ Projects"
permalink: docker-cpp-en
tags: C++ Docker GitLab
---

*Translated from the [Russian original](https://alexprivalov.github.io/docker-cpp) (March 2018) and lightly edited. The talk version was presented at [C++ CoreHard Spring 2018](https://www.youtube.com/watch?v=8hLS1lGk8Fc).*

<img src='/assets/docker-c++/cplus-docker.jpg' style="width: 720px;">
Docker as the quiet centerpiece of development and Continuous Integration across many target platforms.

---

**TL;DR**

- **Problem:** many embedded Linux targets (ARM, MIPS, SuperH, x86-64), each with its own toolchain, vendor SDK, and patched Qt and WebKit/Chromium builds. Setting up a fresh machine to build and run a project took about **two days**.
- **Solution:** one Docker image per platform, distributed through a private registry, started by a small wrapper script, with GitLab CI building both the projects and the images themselves.
- **Result:** a new machine is ready in about **15 minutes** (install Docker once, pull the platform image). Platform developers can patch the SDK and experiment freely, because a broken environment is just a container to recreate. Broken builds on rarely touched platforms are caught by CI instead of by the next unlucky developer.

**Important:** everything below comes from my experience with embedded Linux C/C++ projects. Please keep that in mind while reading.

## Contents
- [The problem](#problem)
- [The solution](#solution)
    - [One container per platform](#docker)
    - [Docker Registry](#registry)
    - [A wrapper script for running Docker images](#wrapper_script)
    - [Containers and Continuous Integration (GitLab-based)](#docker_and_ci)
    - [Automatic image builds](#docker_autobuilds)
- [Downsides and how we dealt with them](#docker_minuses)
- [Results](#resume)
- [Looking back from 2026](#retrospective)

## The problem    {#problem}

Our setup looked like this:

 - N target platforms: different Linux and gcc versions, different CPUs ([ARM](https://en.wikipedia.org/wiki/ARM_architecture_family), [MIPS](https://en.wikipedia.org/wiki/MIPS_architecture), [SuperH](https://en.wikipedia.org/wiki/SuperH), x86-64), and therefore different cross-compilation toolchains;
 - each CPU comes with its own SDK from the chipmaker, in several versions;
 - bugs are found in the SDKs and get patched (sometimes in ways visible to SDK users, since interfaces may change), so the SDK has to be rebuilt and the binaries and headers shared among developers;
 - the same goes for libraries built on top of the SDK, such as Qt and WebKit/Chromium;
 - most developers don't need, or want, to set up and update all of this. They want to land in a ready-made environment and write code;
 - the whole zoo of platforms has to be tested, preferably automatically, at least to check that the code still compiles everywhere.

How much time does a new hire spend building and running a project they've never seen? Cloning the repository, installing toolchains, IDE and libraries, putting everything in the right places, configuring make/cmake/qmake, building, then deploying and running. Along the way a dependency turns out to be missing, a path to a toolchain is wrong, the wrong library version is installed, a symlink is missing, or permissions are off. On a fresh machine this took us about two days, and it usually pulled in colleagues who had solved the same problem six months earlier and no longer remembered how.

Ideally, building a project should look like this:

{% highlight bash %}
    install all required software, SDKs, libs
    git clone git@project.git
    cd project_dir
    make/cmake/build_project.sh
    deploy
    run
{% endhighlight %}

The options I had seen or tried:

 - **everyone sets up their own machine:** slow, not reproducible ("works on my machine"), error-prone, and every new SDK version means everyone repeats the ordeal. A setup script helps only partly;
 - **a shared virtual machine** prepared by the build master and copied by everyone: every SDK update or patch means a new multi-gigabyte VM;
 - **shared build servers:** everyone logs in over ssh and mounts their sources locally via samba/nfs/sshfs. This is what we had. Everyone had the same environment, and a new developer only needed an account. But it was slow: the projects are big, the IDE froze on network operations, and someone was always compiling on the server, so you waited for your build while your own quad-core i7 with 16 GB of RAM sat idle. When the network or the server went down, nobody could work, and one careless change to the environment broke it for everyone;
 - **[Linux containers](https://en.wikipedia.org/wiki/OS-level_virtualization)**, which is what the rest of this post is about.

## The solution  {#solution}

### One container per platform  {#docker}

The solution evolved into containerizing the environment: one platform, one container with everything it needs. Every developer copies it to their machine and starts working right away.

That immediately raised the question of distribution. Carrying images around on a USB stick is no fun, and however many times you write "everybody update!" in the team chat, people will put it off. We needed central distribution, ideally automatic and invisible to developers. Of the options available at the time (LXC, [OpenVZ](https://openvz.org/), [Docker](https://www.docker.com/)), only Docker with its registry infrastructure covered everything.

After the switch, our setup looked like this:

 - every developer works locally, independent of the network;
 - each platform has two "latest" images: `latest` and `latest_debug`. `latest_debug` keeps everything produced while building the SDK and libraries. `latest` is the same image with temporary files removed: object files, archives, library sources, `.git` folders and anything else that application developers don't need;
 - **application developers** build their projects in the `latest` image;
 - **platform developers** patch the platform itself (Linux, the chipmaker's SDK, Qt, WebKit/Chromium, Busybox and so on) in `latest_debug`. Breaking something is no longer scary: in the worst case you throw the container away and start again from the original state. You can experiment without fear of damaging a complex, multi-stage, already-configured build of the whole "world". Because everything in `latest_debug` is already built, a change only recompiles the delta, so you don't wait for all of Qt 5 to rebuild (with a correct cross-compilation configuration!);
 - **CI** uses the `latest` images. Some platforms are always in active development while others are in maintenance mode, where developers rarely go. Before CI, we kept finding the build broken on a rarely touched platform, and had to fix that before starting any real work. With CI, whoever broke the build gets an email from GitLab right away;
 - images are distributed through a private [Docker registry](#registry). Everyone pulls images only from there, and new versions are published there.

### Docker Registry {#registry}

A registry is a server that stores and distributes Docker images. Without one, every developer has to download and load images by hand:

    wget https://cloud.company.com/dockers/specific_img_version.tar
    docker load specific_img_version.tar
    rm -f specific_img_version.tar

With a registry, the Docker client pulls the image itself.

[Docker Hub](https://hub.docker.com/) is the obvious hosted option and takes maintenance off your hands. It didn't work for us: company policy didn't allow storing third-party SDK components outside the company. So we ran our own [registry](https://distribution.github.io/distribution/). It has no web interface; you talk to it through a REST API. For example, to list repositories with curl:

{% highlight bash %}
# list all repositories
curl -X GET https://hub.docker.company_name.com.ua:5000/v2/_catalog
# list image tags in repository repo1
curl -X GET https://hub.docker.company_name.com.ua:5000/v2/repo1/mag324/tags/list
# list image tags in repository repo2
curl -X GET https://hub.docker.company_name.com.ua:5000/v2/repo2/mag351/tags/list
{% endhighlight %}

The registry itself ships as a Docker image and starts like this:

{% gist 5c1c2c57cd0b33feecf32385a414fdf8 run_docker_registry.sh %}

where:

 - **\-\-restart=always** restarts the container if it stops for any reason (the process crashed, the host rebooted);
 - **-v /docker_registry:/var/lib/registry** mounts the server folder that stores all pushed images into the container;
 - **-v /docker_registry/certs:/certs** mounts the certificates, which are not stored inside the container;
 - **REGISTRY_HTTP_TLS_CERTIFICATE** and **REGISTRY_HTTP_TLS_KEY** point to the certificate and the key inside the container.

See the [deployment guide](https://distribution.github.io/distribution/about/deploying/) for details.

**Note:** the Docker client uses https by default. If the registry's domain has no TLS certificate, every host running the Docker client must list it as an [insecure registry](https://distribution.github.io/distribution/about/insecure/) in `/etc/docker/daemon.json`:

    {
      "insecure-registries" : ["registry_ip_without_ssl.com:5000"]
    }

By default the Docker client talks to Docker Hub, so for any other registry the image name must include the registry address. Our naming scheme:

    hub.company.com:5000/chipmaker_name/platform_name:latest

where `hub.company.com:5000` is the registry host and port (if omitted, requests go to Docker Hub). The rest of the name is up to you.

The workflow:

 - an image with everything needed for a platform is built on the build master's machine (later, on CI), validated, and pushed:
{% highlight bash %}
docker push registry_ip:registry_port/image_name:image_version
{% endhighlight %}
 - everyone who needs it pulls the image and works in the ready-made environment:
{% highlight bash %}
docker pull registry_ip:registry_port/image_name:image_version
{% endhighlight %}
 - the previous local version is deleted **by IMAGE ID, not by name**, because one image can have several names, but the ID is unique:
{% highlight bash %}
docker rmi e90ae3da554e
{% endhighlight %}

Running it:
{% highlight bash %}
docker run -it --rm -v /path/to/project/sources:/path/of/sources/inside/container registry_ip:registry_port/image_name:image_version bash
{% endhighlight %}
You get a normal shell, where you go to the project folder and run make/cmake/build.sh. You can also start the build directly:

{% highlight bash %}
docker run -it --rm -v /path/to/project/sources:/path/of/sources/inside/container registry_ip:registry_port/image_name:image_version bash -c "cd /path/of/sources/inside/container; make/cmake/build.sh;"
{% endhighlight %}
This command also plugs into any IDE that supports custom build steps.

All of this works, but most developers don't need or want to know it. So we wrote a simple bash wrapper around Docker.

### A wrapper script for running Docker images {#wrapper_script}

 - the same script everywhere, for developers and for CI;
 - lives in git alongside the project;
 - takes care of the following:
    - no learning curve: you don't need to know Docker to use it;
    - knows which image to pull for the required platform, and from where;
    - creates and starts a new container;
    - mounts the project folder into the container;
    - creates a user matching the host user on the fly, so files produced by the build don't end up owned by root (processes in a container run as root by default);
    - sets up git to work correctly inside the container;
    - removes dangling layers, images and stopped containers so they don't eat disk space;
 - anyone who needs more uses Docker directly, with its mile-long commands :)

The script:
{% gist 3624e6183393dbbb6f7ee251e1e04f8d docker_wrapper.sh %}

Usage:
{% highlight bash %}
cd project_dir
./dockerBuild.sh platformName mode path_to_project optional_cmd
{% endhighlight %}
where:

 - **platformName** is the platform to build for;
 - **mode** is either `dev` or `CI`. `dev` gives you a shell in the project folder, where you can build, work with git and so on. `CI` starts the build automatically;
 - **path_to_project** is the path to the project, here `.` (the current directory);
 - **optional_cmd** is an optional command to run in the container before exiting (`dev` mode only).

### Containers and Continuous Integration (GitLab-based) {#docker_and_ci}

Once the environment could be started on any Linux machine in minutes, Continuous Integration was the obvious next step. We already had GitLab and the Docker registry. The missing piece was a [GitLab Runner](https://docs.gitlab.com/runner/) to build each project for every platform in images pulled from the registry:
![](/assets/docker-c++/slide1.png)
<span class="signed-image">What happens when a developer pushes to the project repository</span>

Later, nightly tests on the current code were added to the same Runner.

### Automatic image builds {#docker_autobuilds}

Once Docker images were part of the daily routine, building new versions of them became the bottleneck: every new environment release meant repeating the same steps by hand. The answer was CI for the environments themselves:

 - a repository with [Dockerfiles](https://docs.docker.com/reference/dockerfile/), bash scripts and configs for building each platform's image;
 - a repository with SDK patches, structured roughly like this:

{% highlight bash %}
Patches_repo
    |-->Platform1
    |       |-->SDKv1
    |       |     |-->01_patch_name.patch
    |       |     |-->02_patch_name.patch
    |       |     |-->.....
    |       |     |-->nn_patch_name.patch
    |       |-->SDKv1.5
    |             |-->01_patch_name.patch
    |             |-->02_patch_name.patch
    |             |-->.....
    |             |-->nn_patch_name.patch
    |-->Platform2
    |       |-->SDKv1
    |       |     |-->...
    |       |-->SDKv1.7
    |             |-->...
    |-->....
    |-->PlatformN
            |-->...
{% endhighlight %}

 - the original SDKs from the chipmakers;
 - any change in the patches repository means the image needs updating, because everyone should have the latest SDK;
 - a [pipeline](https://docs.gitlab.com/ci/pipelines/) runs on every push to the patches repository and applies the patches to clean SDKs in a container. If `patch` fails, the author of the change gets an email with <s>threats</s> errors and fixes it;
 - a [scheduled pipeline](https://docs.gitlab.com/ci/pipelines/schedules/) runs at midnight, checks what changed since the last build, and if anything did, builds the image for the affected platforms, validates it (does the project build in the container? does it run on the target?), pushes it to the registry and sends a notification;
 - in the morning, developers run `dockerBuild.sh` as usual, which pulls the fresh image and deletes the old one.

![](/assets/docker-c++/slide2.png)
<span class="signed-image">How a Docker image gets built</span>

## Downsides and how we dealt with them {#docker_minuses}

 - **Root-owned files.** Processes in a container run as root by default, so files generated in the container can only be changed on the host by root. Solution: the wrapper creates a user on the fly with the same id as the host user.
 - **Weak IDE integration**, partly solved:
    - builds are started through the wrapper script, which is easy to add to the project settings;
    - the IDE can't see inside the container, so it can't parse the SDK headers. The semi-hacky fix: copy the needed headers to the host. Often that's enough.
 - **Disk space** disappears quickly when images are updated often. Solved with a couple of commands in the launch script:

{% highlight bash %}
# force-remove dangling layers and stopped containers
if docker images -f "dangling=true" | grep specific_img --quiet; then
    docker rmi -f $(docker images -f "dangling=true" -q)
fi
{% endhighlight %}

 - **Hard to compare two images** of the same platform built on different dates. Partly solved with a version file inside the image: build date, git hashes and a few other details. The brute-force option is to copy both images' contents to the host and compare them with diff or meld: slow and heavy, but thorough.
 - **Administering the registry is painful.** There is no web UI. Listing repositories and images works only through the REST API, and deleting an outdated image to free space isn't in the API at all, so hacks are unavoidable. GitLab has a [built-in container registry](https://docs.gitlab.com/user/packages/container_registry/) that solves this.
 - **Access control.** I couldn't quickly set up the typical scenario of anonymous pull and authenticated push ([discussion on GitHub](https://github.com/distribution/distribution/issues/1028)). At the time there were two options:
    - HTTP basic auth, where all authenticated users have equal rights, which wasn't what we needed;
    - auth through [nginx](https://distribution.github.io/distribution/recipes/nginx/) or [Apache](https://distribution.github.io/distribution/recipes/apache/), which needed another attempt.

## Results {#resume}

 - **Setting up a fresh machine dropped from about two days to about 15 minutes:** install Docker once, then pull the image for the target platform.
 - **Platform developers stopped being afraid of breaking things.** They could patch the SDK and experiment freely, and if something went wrong, simply recreate the container.
 - Switching between environment versions became easy, so moving "back and forth in time" stopped being a project of its own.
 - The central build servers were freed up, and the idle power of developers' own machines was put to use.
 - SDK and environment builds became automated and repeatable.
 - CI covered both the main projects and the environment images.

The onboarding steps for a new hire are now:
 - `git clone <repo_url> && cd project_dir`
 - `./dockerBuild.sh platformName mode path_to_project optional_cmd`
 - run or deploy the application.

## Looking back from 2026 {#retrospective}

Some of the workarounds above have simpler answers now, and we adopted several of them soon after this post:

 - **GitLab Container Registry** replaced our self-hosted registry, which removed the administration and access-control problems.
 - **Multi-stage builds and BuildKit** made producing the slim `latest` image from the full build a natural part of the Dockerfile.
 - **`docker run --user $(id -u):$(id -g)`** is the simpler way to avoid root-owned files, instead of creating a user inside the container on the fly.

Slides from my talk at C++ CoreHard Spring 2018 ([video](https://www.youtube.com/watch?v=8hLS1lGk8Fc), in Russian):
<iframe width="720" height="405" src="https://docs.google.com/presentation/d/1i_l1pz-J5b6hGABpa1C5a9kxagLW3GU20m1xrQ3zQ2g/embed" frameborder="0"></iframe>
