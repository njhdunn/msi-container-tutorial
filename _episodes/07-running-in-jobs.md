---
title: "Running jobs using Apptainer containers"
teaching: 30
exercises: 40
questions:
- "How do I set up and run a SLURM job from a Apptainer container?"
- "How do I set up and run an parallel MPI job from a Apptainer container?"
objectives:
- "Learn how MPI applications within Apptainer containers can be run on HPC platforms"
- "Understand the challenges and related performance implications when running MPI jobs via Apptainer"
keypoints:
- "Apptainer images containing MPI applications can be built on one platform and then run on another (e.g. an HPC cluster) if the two platforms have compatible MPI implementations."
- "When running an MPI application within a Apptainer container, use the MPI executable on the host system to launch an Apptainer container for each process."
- "Think about parallel application performance requirements and how where you build/run your image may affect that."
---

## Running SLURM jobs that use Apptainer containers

In the most basic case, including Apptainer in your SLURM job won't look very different than what we have been doing interactively so far. For completeness, and so we can see it all at once, let's look at a SLURM batch script that will run the `nhmmer` image we created earlier.

```bash
#!/bin/bash      
#SBATCH --time=00:30:00
#SBATCH --ntasks=1
#SBATCH --mem=10g
#SBATCH --tmp=10g

module load singularity
apptainer exec ~/hmmerInUbuntu.sif nhmmer -h
```

Things can get a little more complicated for workflows that deviate from this pattern, so let's take a look at MPI parallel workflows below, and instance-based workflows in the next episode.

## Running MPI parallel codes with Apptainer containers

### MPI overview

MPI - [Message Passing Interface](https://en.wikipedia.org/wiki/Message_Passing_Interface) - is a widely used standard for parallel programming. It is used for exchanging messages/data between processes in a parallel application. If you've been involved in developing or working with computational science software, you may already be familiar with MPI and running MPI applications.

When working with an MPI code on a large-scale cluster, a common approach is to compile the code yourself, within your own user directory on the cluster platform, building against the supported MPI implementation on the cluster. Alternatively, if the code is widely used on the cluster, the platform administrators may build and package the application as a module so that it is easily accessible by all users of the cluster.

### MPI codes with Apptainer containers

If our target platform uses [OpenMPI](https://www.open-mpi.org/), one of the two widely used source MPI implementations, we can build/install a compatible OpenMPI version within the image as part of the image build process. We can then build our code that requires MPI, either interactively in an image sandbox or via a definition file.

If the target platform uses a version of MPI based on [MPICH](https://www.mpich.org/), the other widely used open source MPI implementation, there is [ABI compatibility between MPICH and several other MPI implementations](https://www.mpich.org/abi/). In this case, you can build MPICH and your code within an image sandbox or as part of the image build process via a definition file, and you should be able to successfully run containers based on this image on your target cluster platform.

MSI has both OpenMPI and MPICH options available, so the best choice here will depend on your specific workflow and the aparallel code you want to run.

As described in Apptainer's [MPI documentation](https://apptainer.org/docs/user/1.0/mpi.html), support for both OpenMPI and MPICH is provided. Instructions are given for building the relevant MPI version from source via a definition file and we'll see this used in an example below.

#### **Container portability and performance on HPC platforms**

While building a container on one system that is intended for use on another, remote HPC platform does provide some level of portability, if you're after the best possible performance, it can present some issues. The version of MPI in the container will need to be built and configured to support the hardware on your target platform if the best possible performance is to be achieved. Where a platform has specialist hardware with proprietary drivers, building on a different platform with different hardware present means that building with the right driver support for optimal performance is not likely to be possible. This is especially true if the version of MPI available is different (but compatible). Apptainer's [MPI documentation](https://apptainer.org/docs/user/1.0/mpi.html) highlights two different models for working with MPI codes. The _[hybrid model](https://apptainer.org/docs/user/1.0/mpi.html#hybrid-model)_ that we'll be looking at here involves using the MPI executable from the MPI installation on the host system to launch apptainer and run the application within the container. The application in the container is linked against and uses the MPI installation within the container which, in turn, communicates with the MPI daemon process running on the host system. In the following section we'll look at building am Apptainer image containing a small MPI application that can then be run using the hybrid model.

### Building and running an Apptainer image for an MPI code

#### **Building and testing an image**

We'll build an image from a definition file. Containers based on this image will print a 'Hello world" message based on the available parallel resources.

Begin by creating a file called `mpitest.c` in your current directory with the contents:

~~~c
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>

int main (int argc, char **argv) {
        int rc;
        int size;
        int myrank;

        rc = MPI_Init (&argc, &argv);
        if (rc != MPI_SUCCESS) {
                fprintf (stderr, "MPI_Init() failed");
                return EXIT_FAILURE;
        }

        rc = MPI_Comm_size (MPI_COMM_WORLD, &size);
        if (rc != MPI_SUCCESS) {
                fprintf (stderr, "MPI_Comm_size() failed");
                goto exit_with_error;
        }

        rc = MPI_Comm_rank (MPI_COMM_WORLD, &myrank);
        if (rc != MPI_SUCCESS) {
                fprintf (stderr, "MPI_Comm_rank() failed");
                goto exit_with_error;
        }

        fprintf (stdout, "Hello, I am rank %d/%d\n", myrank, size);

        MPI_Finalize();

        return EXIT_SUCCESS;

 exit_with_error:
        MPI_Finalize();
        return EXIT_FAILURE;
}
~~~

In the same directory, save the following definition file content to a `.def` file, e.g. `ompi_example.def`:

~~~
Bootstrap: docker
From: ubuntu:20.04

%files
    mpitest.c /opt

%environment
    # Point to OMPI binaries, libraries, man pages
    export OMPI_DIR=/opt/ompi
    export PATH="$OMPI_DIR/bin:$PATH"
    export LD_LIBRARY_PATH="$OMPI_DIR/lib:$LD_LIBRARY_PATH"
    export MANPATH="$OMPI_DIR/share/man:$MANPATH"

%post
    echo "Installing required packages..."
    apt update && apt install -y wget rsh-client build-essential

    echo "Installing Open MPI"
    export OMPI_DIR=/opt/ompi
    export OMPI_VERSION=4.1.5
    export OMPI_URL="https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-$OMPI_VERSION.tar.bz2"
    mkdir -p /tmp/ompi
    mkdir -p /opt
    # Download
    cd /tmp/ompi && wget -O openmpi-$OMPI_VERSION.tar.bz2 $OMPI_URL && tar -xjf openmpi-$OMPI_VERSION.tar.bz2
    # Compile and install
    cd /tmp/ompi/openmpi-$OMPI_VERSION && ./configure --prefix=$OMPI_DIR && make -j8 install

    # Set env variables so we can compile our application
    export PATH=$OMPI_DIR/bin:$PATH
    export LD_LIBRARY_PATH=$OMPI_DIR/lib:$LD_LIBRARY_PATH

    echo "Compiling the MPI application..."
    cd /opt && mpicc -o mpitest mpitest.c
    cp mpitest /bin
~~~
{: .output}

A quick overview of what the above definition file is doing:

 - The image is being bootstrapped from the `ubuntu:20.04` Docker image.
 - In the `%files` section: The MPI "Hello world" test is copied from the current directory into the `/opt` directory within the image.
 - In the `%environment` section: Set a couple of environment variables that will be available within all containers run from the generated image.
 - In the `%post` section:
   - Ubuntu's `apt` package manager is used to update the package directory and then install the compilers and other libraries required for the OMPI build.
   - The OMPI .tar.bz2 file is extracted and the configure, build and install steps are run. 
   - The environment is set up to use the newly installed OMPI, and we build our "Hello world" example.

> ## Build and test the image
>
> Using the above definition file, build an Apptainer image named `ompi_example.sif`.
> 
> Once the image has finished building, test it by running the `mpitest` program that we built.
> 
> > ## Solution
> > 
> > You should be able to build an image from the definition file as follows:
> > 
> > ~~~
> > $ apptainer build ompi_example.sif ompi_example.def
> > ~~~
> > {: .language-bash}
> >
> >
> > 
> > Let's begin with a single-process run of `mpitest` to ensure that we can run the container as expected. We'll use the MPI installation _within_ the container for this test. _Note that when we run a parallel job on an HPC cluster platform, we use the MPI installation on the cluster to coordinate the run so things are a little different..._
> > 
> > Start a shell in the Apptainer container based on your image and then run a single process job via `mpirun`:
> > 
> > ~~~
> > $ apptainer shell ompi_example.sif
> > Apptainer> mpirun -np 1 mpitest
> > ~~~
> > {: .language-bash}
> > 
> > You should see output similar to the following:
> > 
> > ~~~
> > Hello, I am rank 0/1
> > ~~~
> > {: .output}
> {: .solution}
{: .challenge}

#### **Running Apptainer containers via MPI**

Assuming the above tests worked, we can now try undertaking a parallel run within our container image.

This is where things get interesting and we'll begin by looking at how Apptainer containers are run within an MPI environment.

If you're familiar with running MPI codes, you'll know that you use `mpirun` (as we did in the previous example), `mpiexec` or a similar MPI executable to start your application. This executable may be run directly on the local system or cluster platform that you're using, or you may need to run it through a job script submitted to a job scheduler. Your MPI-based application code, which will be linked against the MPI libraries, will make MPI API calls into these MPI libraries which in turn talk to the MPI daemon process running on the host system. This daemon process handles the communication between MPI processes, including talking to the daemons on other nodes to exchange information between processes running on different machines, as necessary.

When running code within an Apptainer container, we don't use the MPI executables stored within the container (i.e. we DO NOT run `apptainer exec mpirun -np <numprocs> /path/to/my/executable`). Instead we use the MPI installation on the host system to run Apptainer and start an instance of our executable from within a container for each MPI process. Without Apptainer support in an MPI implementation, this results in starting a separate Apptainer container instance within each process. This can present some overhead if a large number of processes are being run on a host. Where Apptainer support is built into an MPI implementation this can address this potential issue and reduce the overhead of running code from within a container as part of an MPI job.

Ultimately, this means that our running MPI code is linking to the MPI libraries from the MPI install within our container and these are, in turn, communicating with the MPI daemon on the host system which is part of the host system's MPI installation. In the case of OMPI, these two installations of MPI may be different but as long as there is [ABI compatibility](https://wiki.mpich.org/mpich/index.php/ABI_Compatibility_Initiative) between the version of MPI installed in your container image and the version on the host system, your job should run successfully.

We can now try running a 2-process MPI run of our test program. 

> ## Undertake a parallel run of `mpitest` (general example)
> 
> You should be able to run the example using a command similar to the one shown below. However, if you are not currently inside an interactive SLURM job, you may need to write and submit a job submission script at this point to initiate running of the benchmark.
> 
> Also note that due to a peculiarity of how we install OMPI at MSI, we will need to unset OPAL_PREFIX before running a hybrid OMPI+Apptainer job.
>
> ~~~
> unset OPAL_PREFIX
> $ mpirun -np 2 apptainer exec ompi_example.sif mpitest
> ~~~
> {: .language-bash}
> 
> > ## Expected output and discussion
> > 
> > As you can see in the mpirun command shown above, we have called `mpirun` on the host system and are passing to MPI the `apptainer` executable for which the parameters are the image file and any parameters we want to pass to the image's run script, in this case the path/name of the executable to run.
> > 
> > 
> >~~~
> > Hello, I am rank 1/2
> > Hello, I am rank 0/2
> > ~~~
> > {: .output}
> {: .solution}
{: .challenge}


