---
title: "Apptainer/Singularity: Getting started"
start: true
teaching: 30
exercises: 20
questions:
- "What is Apptainer/Singularity and why might I want to use it?"
- "What issues motivated the creation of Apptainer/Singularity?"
- "What are the differences between Docker, Apptainer and Singularity?"
objectives:
- "Understand what Apptainer/Singularity is and when you might want to use it."
- "Learn the design goals behind Apptainer/Singularity."
keypoints:
- "Apptainer/Singularity is a container platform designed by and for scientists."
- "Apptainer/Singularity has a different security model to other container platforms, one of the key reasons that it is well suited to HPC and cluster environments. User inside the container = user outside."
- "Apptainer/Singularity has its own container image format (SIF)."
---


# Working with containers

> ## Images and containers
> We'll start with a brief note on the terminology used in this section of the course. We refer to both **_images_** and **_containers_**. What is the distinction between these two terms? 

> **_Images_** are bundles of files including an operating system, software and potentially data and other application-related files. They may sometimes be referred to as a _disk image_ or _container image_ and they may be stored in different ways, perhaps as a single file, or as a group of files. Either way, we refer to this file, or collection of files, as an image.

> A **_container_** is a virtual environment that is based on an image. That is, the files, applications, tools, etc that are available within a running container are determined by the image that the container is started from. It may be possible to start multiple container instances from an image. You could, perhaps, consider an image to be a form of template from which running container instances can be started.
{: .callout}

Containers are packages of software that encapsulates a system environment. An OS-level virtualization is delivered in a container, and any program running on it will use the contextualization isolated inside the container. They have the advantage that you can build a container on any system, your laptop for example, and then execute it anywhere as far as the platform compatible with the container is available.

Concepts such as reproducibility, preservation, and distribution are important in the HPC community, and the containers provide a solution totally compatible with such concepts:

- The version of some specific software used to perform an analysis can be preserved in a container with exactly the same environment used at that time.
- A "legacy software" with binaries only available for an outdated OS can be executed inside a container.
- All the necessary packages to process data can be easily distributed in containers, independently of the operating system available on the sites.


> ## What about virtual machines?
> Virtual Machines (VMs) provide the same isolation and reproducibility. However, they emulate the hardware, so they are computationally heavier to run, require bigger files when distributed and are less flexible than containers, that run only what you require to be different.
{: .callout}


# Apptainer/Singularity
Apptainer/Singularity is a container platform that allows software engineers and researchers to easily share their work with others by packaging and deploying their software applications in a portable and reproducible manner. When you download a Singularity container image, you essentially receive a virtual computer disk that contains all of the necessary software, libraries and configuration to run one or more applications or undertake a particular task, e.g. to support a specific research project. This saves you the time and effort of installing and configuring software on your own system or setting up a new computer from scratch, as you can simply run a Singularity container from the image and have a virtual environment that is identical to the one used by the person who created the image. Container platforms like Singularity provide a convenient and consistent way to access and run software and tools. Singularity is increasingly widely used in the research community for supporting research projects as it allows users to isolate their software environments from the host operating system and can simplify tasks such as running multiple experiments simultaneously.

Many solutions are available to work with containers, for example Docker, one of the most popular platforms. However, the enterprise-based container frameworks were motivated to provide micro-services, a solution that fits well in the models of the industry, where system administrators with root privilege install and run applications, each in its own container. This is not so compatible with the workflow in the High-Performance Computing (HPC) and High Throughput Computing (HTC), in which usually complex applications run exhaustively using all the available resources and without any special privilege.

Apptainer/Singularity is a container platform created for the HPC/HTC use case. It allows users to build and run containers with just a few steps in most of the cases, and its design presents key concepts for the scientific community:

- Single-file based container images, facilitating the distribution, archiving and sharing.
- Ability to run, and in modern systems also to be installed, without any root daemon or setuid privileges. This makes it safer for large computer centers with shared resources.
- Preserves the permissions in the environment. The user outside the container can be the same user inside.
- Simple integration with resource managers and distributed computing frameworks because it runs as a regular application.

## Apptainer vs Singularity
In these lessons you see the name *Apptainer* or *Apptainer/Singularity*, and the command `apptainer`.
As stated in the [move and renaming announcement](https://apptainer.org/news/community-announcement-20211130/), "Singularity IS Apptainer".
Currently there are three products derived from the original Singularity project from 2015:
* *Singularity*: commercial software by [Sylabs](https://sylabs.io/).
* [*SingularityCE*](https://sylabs.io/2022/06/singularityce-is-singularity/): open source Singularity supported by Sylabs.
* *Apptainer*: open source Singularity, recently renamed and hosted by the [Linux Foundation](https://www.linuxfoundation.org/).
As of Fall 2022 all three Apptainer/Singularity versions are compatible and practically the same, but have different roadmaps.
There is hope that in the future they will join forces, but this is not currently the case.
To understand how this came to be you can read the [Singularity history on Wikipedia](https://en.wikipedia.org/wiki/Singularity_%28software%29#History).

MSI provides Apptainer, the most adopted variation in the scientific community, so we are using the `apptainer` command.
If you are using Singularity or SingularityCE, just replace the command `apptainer` with `singularity` and the
`APPTAINER_` and  `APPTAINERENV_` variable prefixes  with `SINGULARITY_` and  `SINGULARITYENV_`.
 But since its previous version was named Singularity and the developers wanted backwards-compatibility, if you have older scripts still using the `singularity` command they will work also in Apptainer because it is providing the `singularity` alias
and [full compatibility with the previous Singularity environment](https://apptainer.org/docs/user/main/singularity_compatibility.html).

## Documentation
The [official Apptainer documentation](https://apptainer.org/docs/) is available online. Contains basic and advanced
usage of Apptainer/Singularity beyond the scope of this training document. Take a look and read the nice
[introduction](https://apptainer.org/docs/user/main/introduction.html), explaining the motivation behind the
creation of Apptainer/Singularity.


