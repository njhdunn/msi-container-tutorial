---
title: "Building Containers"
teaching: 40
exercises: 30
questions:
- "How to build containers with my requirements?"
objectives:
- "Download and assemble containers from available images in the repositories."
keypoints:
- "The command `build` is the basic tool for the creation of containers."
- "A _sandbox_ is a writable directory where containers can be built interactively."
- "Superuser permissions are required to build containers if you need to install packages or manipulate the operating system."
- "Use interactive builds only for development and tests, use definition files for production or publicly distributed containers."
---

Running containers from the available public images is not the only option. In many cases, it is required to modify
an image or even to create a new one from scratch. For such purposes, Apptainer provides the command `build`,
defined in the documentation as the _Swiss army knife_ of container creation.

The usual workflow is to prepare and test a container in a build environment (like your laptop),
either with an interactive session or from a definition file,
and then to deploy the container into a production environment for execution (as your institutional cluster).
Interactive sessions are great to experiment and test your new container.
If you want to distribute the container or use it in production, then we recommend to build it from a definition file, as described in the next episode.
This ensures the greatest possibility of reproducibility and transparency.

<figure>
  <img src="https://journals.plos.org/plosone/article/figure/image?size=large&id=10.1371/journal.pone.0177459.g001" alt="Apptainer/Singularity usage workflow"/>
  <figcaption>'Apptainer/Singularity usage workflow' via <i>Kurtzer GM, Sochat V, Bauer MW (2017) Singularity: Scientific containers for mobility of compute. PLoS ONE 12(5): e0177459. <a href="https://doi.org/10.1371/journal.pone.0177459">https://doi.org/10.1371/journal.pone.0177459</a></i></figcaption>
</figure>

## Build a container in an interactive session

> ## Notes on shared file systems
> Avoid using network storage as a sandbox directory, as these systems can lead to permissions issues and very slow performance. At MSI, this applies to your home directory, group storage, and global scratch.
> Instead, use the local storage available inside a SLURM job via the `--tmp` flag. You can then work out of this directory for the duration of the job, but make sure to copy any results you want to save out of `/tmp` before the end of the job or it will be deleted.
{:.callout}

Let's request some resources from SLURM, so that we have local storage and can run longer build processes than would be possible on our login nodes.
```bash
salloc --time=6:00:00 --nodes=1 --ntasks-per-node=8 --partition=amdsmall,amdlarge --cluster=mesabi --tmp=48gb --mem=16g
```
~~~
salloc: Setting account: dunn0404
salloc: Pending job allocation 160383731
salloc: job 160383731 queued and waiting for resources
salloc: job 160383731 has been allocated resources
salloc: Granted job allocation 160383731
salloc: Waiting for resource configuration
salloc: Nodes cn1122 are ready for job
~~~
{: .output}
```bash
ssh cn1122
cd /tmp
module load singularity
```

While images contained in the `.sif` files are more compact and immutable objects, ideal for reproducibility, for building and testing images in more convenient the use of a _sandbox_, which can be easily modified.

The command `build` provides a flag `--sandbox` that will create a writable directory, `myCentOS7`, in your work directory:
```bash
apptainer build --sandbox myUbuntu docker://ubuntu:20.04
mkdir myUbuntu/users
chmod -R a=rwX myUbuntu
```

The container name is `myUbuntu`, and it has been initialized from the [official Docker image](https://hub.docker.com/_/ubuntu)
of Ubuntu 20.04
We had to do a little extra setup for the sandbox because of some permissions peculiarities, by creating the `users` directory for the sandbox and changing the permissions for the whole sandbox.

To initialize an interactive session use the `shell` command. And to write files within the sandbox directory use the `--writable` option.
Finally, the installation of new components will require superuser access inside the container, so use also the `--fakeroot` option, unless you are already root also outside.
```bash
apptainer shell --writable --fakeroot myUbuntu
Apptainer> whoami
```
~~~
root
~~~
{: .output}
Depending on the Apptainer/Singularity installation (privileged or unprivileged) and the version, you may have some requirements, like the `fakeroot` utility or `newuidmap` and `newgidmap`.
If you get an error when using `--fakeroot` have a look at the [fakeroot documentation](https://apptainer.org/docs/user/main/fakeroot.html). MSI is currently using `fakeroot` for Apptainer on our clusters.
> ## `--fakeroot` is not root
> ATTENTION! [`--fakeroot`](https://apptainer.org/docs/user/main/fakeroot.html) allows you to be root inside a container that you own but is not changing who you are outside.
> All the outside actions and the writing on bound files and directories will happen as your outside user, even if in the container is done by root.
{: .callout}

As an example, let's create a container with hmmer 3.4 available using the `myUbuntu` sandbox.
First, we need to install the development tools (remember that in this interactive session we are superuser):
```bash
Apptainer> apt update
Apptainer> apt install build-essential
Apptainer> apt install wget
```
Where `apt` is the [package manager used in Debian distributions](https://en.wikipedia.org/wiki/APT_(software))
(like Ubuntu).

We will follow the a modified version of the
installation steps described in the [hmmer website](https://hmmer.org/).
Here is a summary of the commands you will need (you may need to adjust link and commands if there is a new Hmmer version):

```bash
Apptainer> mkdir /opt/hmmer && cd /opt/hmmer
Apptainer> wget http://eddylab.org/software/hmmer/hmmer.tar.gz
Apptainer> tar zxf hmmer.tar.gz
Apptainer> ln -s hmmer*/ hmmer
Apptainer> cd hmmer
Apptainer> ./configure
Apptainer> make

Apptainer> exit
```

The last step before we can use the sandbox normally is to update the permissions again, for all of the new files we just installed:
```bash
chmod -R a=rwX myUbuntu
```

Now, open an interactive session with your user (no `--fakeroot`). You can use now the container with hmmer ready in a
few steps. Let's check by printing the help documentation for `nhmmer

```bash
apptainer shell myUbuntu

Apptainer> export PATH=$PATH:/opt/hmmer/hmmer/src
Apptainer> nhmmer -h
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

Options controlling scoring system:
  --singlemx    : use substitution score matrix w/ single-sequence MSA-format inputs
  --popen <x>   : gap open probability  [0.03125]  (0<=x<0.5)
  --pextend <x> : gap extend probability  [0.75]  (0<=x<1)
  --mxfile <f>  : read substitution score matrix from file <f>

Options controlling reporting thresholds:
  -E <x> : report sequences <= this E-value threshold in output  [10.0]  (x>0)
  -T <x> : report sequences >= this score threshold in output

Options controlling inclusion (significance) thresholds:
  --incE <x> : consider sequences <= this E-value threshold as significant  [0.01]  (x>0)
  --incT <x> : consider sequences >= this score threshold as significant
...
~~~
{: .output}

Notice that we need to update the environment variable `PATH` in order to use hmmer.
We will automate this in the next section.

> ## Execute Python with PyROOT available
>
> Build a container to use uproot in Python 3.9.
>
> > ## Solution
> >
> > Start from the [Python 3.9 Docker image](https://hub.docker.com/_/python) and create the `myPython` sandbox:
> > ```bash
> > apptainer build --sandbox myPython docker://python:3.9
> > apptainer shell myPython
> > ```
> > Once inside the container, you can install [Uproot](https://uproot.readthedocs.io/en/latest/index.html).
> > ```bash
> > Apptainer> python3 -m pip install --upgrade pip
> > Apptainer> python3 -m pip install uproot awkward
> > ```
> > Exit the container and use it as you like:
> > ```bash
> > apptainer exec myPython python -c "import uproot; print(uproot.__doc__)"
> > ```
> > ~~~
> > Uproot: ROOT I/O in pure Python and NumPy.
> > ...
> > ~~~
> > {: .output}
> > Notice how we did not need neither `--writable ` nor `--fakeroot` for the installation, but everything worked fine since pip installs user packages in the user $HOME directory.
> > In addition, Apptainer/Singularity by default mounts the user home directory as read+write, even if the container is read-only.
> > This is why a _sandbox_ is great to test and experiment locally, but should not be used for containers that will be shared or deployed. Manual changes and local directories are difficult to reproduce and control. Once you are happy with the content, you should use definition files, described in the next episode.
> {: .solution}
{: .challenge}
