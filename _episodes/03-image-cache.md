---
title: "The image cache"
teaching: 10
exercises: 0
questions:
- "Why does Apptainer use a local cache?"
- "Where does Apptainer store images?"
objectives:
- "Learn about Apptainer's image cache."
- "Learn how to manage Apptainer images stored locally."
keypoints:
- "Apptainer caches downloaded images so that an unchanged image isn't downloaded again when it is requested using the `apptainer pull` command."
- "You can free up space in the cache by removing all locally cached images or by specifying individual images to remove."
---

## Apptainer's image cache

While Apptainer doesn't have a local image repository in the same way as, for instance, Docker, it does cache downloaded image files. As we saw in the previous episode, images are simply `.sif` files stored on your local disk. 

If you delete a local `.sif` image that you have pulled from a remote image repository and then pull it again, if the image is unchanged from the version you previously pulled, you will be given a copy of the image file from your local cache rather than the image being downloaded again from the remote source. This removes unnecessary network transfers and is particularly useful for large images which may take some time to transfer over the network. To demonstrate this, remove the `hello-world.sif` file stored in your `test` directory and then issue the `pull` command again:

~~~
$ rm centos7-devel_latest.sif
$ apptainer pull library://gmk/default/centos7-devel
~~~
{: .language-bash}

~~~
INFO:    Use image from cache
~~~
{: .output}

As we can see in the above output, the image has been returned from the cache and we don't see the output that we saw previously showing the image being downloaded from the Container Library.

How do we know what is stored in the local cache? We can find out using the `apptainer cache` command:

~~~
$ apptainer cache list
~~~
{: .language-bash}

~~~
There are 2 container file(s) using 953.56 MiB and 7 oci blob file(s) using 795.96 MiB of space
Total space used: 1.71 GiB
~~~
{: .output}

This tells us how many container files are stored in the cache and how much disk space the cache is using but it doesn't tell us _what_ is actually being stored. To find out more information we can add the `-v` verbose flag to the `list` command:

~~~
$ apptainer cache list -v
~~~
{: .language-bash}

~~~
NAME                     DATE CREATED           SIZE             TYPE
3153aa388d026c26a2235e   2023-11-29 11:43:11    28.17 MiB        blob
4f4fb700ef54461cfa0257   2023-11-29 11:43:10    0.03 KiB         blob
a92e1499ab2116def52960   2023-11-29 11:43:20    1.01 KiB         blob
adcfc5ae21e02d2a4e0611   2023-11-29 11:43:20    3.99 KiB         blob
cd4f73f3be7df86c541481   2023-11-29 11:43:20    600.84 MiB       blob
e9d9ea00e81a9ebf9a6a5d   2023-11-29 11:43:13    166.94 MiB       blob
fc99e5b541a5d800e7a738   2023-11-29 11:43:11    0.37 KiB         blob
sha256.740fa5a3d1a2019   2023-11-29 11:42:32    296.19 MiB       library
3dd56175ed0c777d5dda16   2023-11-29 11:44:39    657.37 MiB       oci-tmp

There are 2 container file(s) using 953.56 MiB and 7 oci blob file(s) using 795.96 MiB of space
Total space used: 1.71 GiB
~~~
{: .output}

This provides us with some more useful information about the actual images stored in the cache. In the `TYPE` column we can see that our image type is `library` because it's a `SIF` image that has been pulled from the Container Library. 

> ## Cleaning the Apptainer image cache
> We can remove images from the cache using the `apptainer cache clean` command. Running the command without any options will display a warning and ask you to confirm that you want to remove everything from your cache.
>
> You can also remove specific images or all images of a particular type. Look at the output of `apptainer cache clean --help` for more information.
{: .callout}

> ## Cache location
> By default, Apptainer uses `$HOME/.apptainer/cache` as the location for the cache. You can change the location of the cache by setting the `APPTAINER_CACHEDIR` environment variable to the cache location you want to use.
>
> The labels in the `TYPE` column of the cache correspond to subdirectories of the Apptainer cache, and you can get more information about a specific entry by using `apptainer inspect` on the a specific file. For instance, if we want to inspect the `library` entry from the above output, we could run:
>
> ~~~
> $ apptainer inspect ~/.apptainer/cache/library/sha256.740fa5a3d1a2019*
> ~~~
> {: .language-bash}
{: .callout}
