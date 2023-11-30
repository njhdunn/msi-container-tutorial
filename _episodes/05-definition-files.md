---
title: "Containers from definition files"
teaching: 40
exercises: 30
questions:
- "How to easily build and deploy containers from a single definition file?"
objectives:
- "Create a container from a definition file."
keypoints:
- "An Apptainer definition file provides an easy way to build and deploy containers."
---

As shown in the previous chapter, building containers with an interactive session may take several steps, and it can
become as complicated as the setup is needed.
An Apptainer definition file provides an easy way to build and deploy containers. The same advice for using local storage applies here, and we will be using the same `/tmp` location that we allocated in the sandbox episode.


## Hello World Apptainer

The following recipe shows how to build a hello-world container, and run the container on your local computer.

- Step 1: Open a text editor (e.g., nano, vim, or gedit in a graphical environment)

  ```bash
  vim hello-world.def
  ```

- Step 2: Include the following script in the `hello-world.def` file to define the environment

  ```bash
  BootStrap: docker
  From: ubuntu:20.04

  %runscript
    echo "Hello World"
  # Print Hello world when the image is loaded
  ```

    In the above script, the first line - `BootStrap: docker` indicates that apptainer will use the docker protocol to retrieve the base OS to start the image.
The `From: ubuntu:20.04` is given to apptainer to start from a specific image/OS in Docker Hub.
Any content within the  `%runscript` will be written to a file that is executed when one runs the apptainer image.
The `echo "Hello World"` command will print the `Hello World` on the terminal.
Finally the `#` hash is used to include comments within the definition file.

- Step 3: Build the image

  ```bash
  apptainer build hello-world.sif hello-world.def
  ```

    The `hello-world.sif` file specifies the name of the output file that is built when using the `apptainer build` command.

- Step 4: Run the image

  ```bash
  ./hello-world.sif
  ```

### Deleting Apptainer image
To delete the hello-world Apptainer image, simply delete the `hello-world.sif` file.

> ## `apptainer delete`
> Note that there is also a `apptainer delete` command, but it is to delete an image from a remote library.
> To learn more about using remote endpoints and pulling and pushing images from or to libraries, read
> [Remote Endpoints](https://apptainer.org/docs/user/main/endpoint.html) and [Library API Registries](https://apptainer.org/docs/user/main/library_api.html).
{: .callout}


## Example of a more elaborate definition file

Let's look at the structure of the definition file with another example. Let's prepare a container from an [official
Ubuntu image](https://hub.docker.com/_/ubuntu), but this time we will install hmmer in an automated way.

Adapting our sandbox procedure to the definition file will look like:

~~~
BootStrap: docker
From: ubuntu:20.04

%post
        apt update && apt install build-essential wget -y

        mkdir /opt/hmmer && cd /opt/hmmer
        wget http://eddylab.org/software/hmmer/hmmer.tar.gz
        tar zxf hmmer.tar.gz
        ln -s hmmer*/ src
        cd src
        ./configure --prefix=/opt/hmmer/install
        make
        make install

%environment
        export PATH=$PATH:/opt/hmmer/install/bin


%runscript
        nhmmer -h

%labels
    Author dunn0404
    Version v0.0.1

%help
    Example container running the nhmmer help documentation
~~~
{: .source}

Let's take a look at the [definition file](https://apptainer.org/docs/user/main/definition_files.html):
* The first two lines define the base image. In this case, the image `ubuntu:20.04` from Docker Hub is used.
* `%post` are lines to execute inside the container after the OS has been set. In this example, we are listing the
steps that we would follow to install ROOT with a precompiled binary in an interactive session.
Notice that the binary used corresponds with the Ubuntu version defined at the second line.
* `%environment` is used to define environment variables available inside the container. Here we are setting the env
variables required to execute ROOT and PyROOT.
* Apptainer containers can be executable. `%runscript` define the actions to take when the container is executed.
To illustrate the functionality, we will just run `nhmmer -h` to print the help text for `nhmmer`.
* `%labels` add custom metadata to the container.
* `%help` it is the container documentation: what it is and how to use it. Can be displayed using `apptainer run-help`

Save this definition file as `hmmerInUbuntu.def`. To build the container, just provide the definition file as argument
(executing as superuser):
```bash
apptainer build hmmerInUbuntu.sif hmmerInUbuntu.def
```

Then, an interactive shell inside the container can be initialized with `apptainer shell`, or
a command executed with `apptainer exec`. A third option is execute the actions defined inside `%runscript`
simply by calling the container as an executable

```bash
./hmmerInUbuntu.sif
```
~~~
# nhmmer :: search a DNA model, alignment, or sequence against a DNA database
# HMMER 3.4 (Aug 2023); http://hmmer.org/
# Copyright (C) 2023 Howard Hughes Medical Institute.
# Freely distributed under the BSD open source license.
# - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Usage: nhmmer [options] <query hmmfile|alignfile|seqfile> <target seqfile>

Basic options:
  -h : show brief help on version and usage

Options directing output:
  -o <f>             : direct output to file <f>, not stdout
  -A <f>             : save multiple alignment of all hits to file <f>
  --tblout <f>       : save parseable table of hits to file <f>
  --dfamtblout <f>   : save table of hits to file, in Dfam format <f>
  --aliscoresout <f> : save scores for each position in each alignment to <f>
  --hmmout <f>       : if input is alignment(s), write produced hmms to file <f>
  --acc              : prefer accessions over names in output
  --noali            : don't output alignments, so output is smaller
  --notextw          : unlimit ASCII text output line width
  --textw <n>        : set max width of ASCII text output lines  [120]  (n>=120)
...
~~~
{: .output}

If we like the results, we need to be sure to save the SIF format image to somewhere permanent, like our home directory. If we don't it will be deleted alongside the other contents of `/tmp` when the job ends.

```bash
cp hmmerInUbuntu.sif ~/
```

Here we have covered the basics with a few examples selected to highlight the Apptainer fundamentals.
Check the [Apptainer docs](https://apptainer.org/docs/user/main/build_a_container.html) to see all the available
options and more details related to the container creation.

A few [best practices for your containers](https://apptainer.org/docs/user/1.0/definition_files.html#best-practices-for-build-recipes) to make them more usable, portable, and secure:
1. Always install packages, programs, data, and files into operating system locations (e.g. not `/home`, `/tmp` , or any other directories that might get commonly binded on).
1. Document your container. If your runscript doesn’t supply help, write a `%help` or `%apphelp` section. A good container tells the user how to interact with it.
1. If you require any special environment variables to be defined, add them to the `%environment` and `%appenv` sections of the build recipe.
1. Files should always be owned by a system account (UID less than 500).
1. Ensure that sensitive files like `/etc/passwd`, `/etc/group`, and `/etc/shadow` do not contain secrets.
1. Build production containers from a definition file instead of a sandbox that has been manually changed. This ensures the greatest possibility of reproducibility and mitigates the “black box” effect.

> ## Deploying your containers
> Keep in mind that, while building a container may be time consuming, the execution can be immediate and anywhere your image is available.
> Once your container is built with the requirements of your analysis, you can deploy it in a large cluster and execute it
> as far as Apptainer is available on the site.
>
> Libraries like [Sylabs Cloud Library](https://cloud.sylabs.io/library) ease the distribution of images.
> Organizations like OSG provide instructions to [use available images](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers/)
and [distribute custom images via CVMFS](https://portal.osg-htc.org/documentation/htc_workloads/using_software/containers-docker/).
>
> Be smart, and this will open endless possibilities in your workflow.
{: .callout}


> ## Write a definition file to build a container with Pythia8 available in Python
>
> Following the example of the first section in which a container is built with an interactive session,
> write a definition file to deploy a container with Pythia8 available.
>
> Take a look at
> [`/opt/pythia/pythia8307/examples/main01.py`](https://gitlab.com/Pythia8/releases/-/blob/pythia8307/examples/main01.py)
> and define the `%runscript` to execute it using `python3`.
>
> (Tip: notice that main01.py requires `Makefile.inc`).
>
> > ## Solution
> > ~~~
> > BootStrap: docker
> > From: centos:centos7
> >
> > %post
> >     yum -y groupinstall 'Development Tools'
> >     yum -y install python3-devel
> >     mkdir /opt/pythia && cd /opt/pythia
> >     curl -o pythia8307.tgz https://pythia.org/download/pythia83/pythia8307.tgz
> >     tar xvfz pythia8307.tgz
> >     cd pythia8307
> >     ./configure --with-python-include=/usr/include/python3.6m/
> >     make
> >
> > %environment
> >     export PYTHONPATH=/opt/pythia/pythia8307/lib:$PYTHONPATH
> >     export LD_LIBRARY_PATH=/opt/pythia/pythia8307/lib:$LD_LIBRARY_PATH
> >
> > %runscript
> >     cp /opt/pythia/pythia8307/Makefile.inc .
> >     python3 /opt/pythia/pythia8307/examples/main01.py
> >
> > %labels
> >     Author HEPTraining
> >     Version v0.0.1
> >
> > %help
> >     Container providing Pythia 8.307. Execute the container to run an example.
> >     Open it in a shell to use the Pythia installation with Python 3.6
> > ~~~
> > {: .source}
> >
> > Build your container executing
> >
> > ```bash
> > apptainer build pythiaInCentos7.sif myPythia8.def
> > ```
> >
> > And finally, execute the container to run [`main01.py`](https://gitlab.com/Pythia8/releases/-/blob/pythia8307/examples/main01.py)
> >
> > ```bash
> > ./pythiaInCentos7.sif
> > ```
> {: .solution}
{: .challenge}
