---
layout: lesson
root: .  # Is the only page that doesn't follow the pattern /:path/index.html
permalink: index.html  # Is the only page that doesn't follow the pattern /:path/index.html
---

This lesson provides an introduction to using the [Apptainer/Singularity container platform](https://apptainer.org). Apptainer/Singularity is particularly suited to running containers on infrastructure where users don't have administrative privileges, for example shared infrastructure such as High Performance Computing (HPC) clusters. 

This lesson will introduce Apptainer/Singularity from scratch showing you how to run a simple container and building up to creating your own containers and running parallel scientific workloads on HPC infrastructure.

<!-- this is an html comment -->

{% comment %} This is a comment in Liquid {% endcomment %}

> ## Prerequisites
> * Basic knowledge of the Unix Shell, e.g., from the [carpentry course](https://swcarpentry.github.io/shell-novice/).
> * Knowledge of using the SLURM scheduler at MSI
> * An active MSI account if you plan to follow along with the examples. 
>
> ** For this training we recommend Apptainer >= 1.0 or Singularity >= 3.5. Older versions may not have some of the features or behave differently
{: .prereq}

{% include links.md %}
