# Dartmouth Genomic Data Science Core - Pixi SOP

**Author: Mike Martinez M.S.**

**Date: December 12, 2025**

**Purpose:** 
To provide a standardized framework for implmenting and managing software environments for core projects using Pixi

**Scope:** 
This SOP applies to any projects where a reproducible environment is needed, including R, Python, command-line tools, and mixed-language workflows. Pixi may not be applicable for every project due to current limitations in support for Bioconductor tools (R) as of December 2025. 



--------

# Table of Contents

- [SOP](#sop)
  - [Initializing the Pixi Environment](#initializing-the-pixi-environment)
  - [Adding Software to the Environment](#adding-software-to-the-environment)
  - [Activating the Environments](#activating-the-environments)
    - [Project Local Shell](#project-local-shell)
    - [Execute Commands Locally](#execute-commands-locally)
    - [Using Pixi Environments with SLURM](#using-pixi-environments-with-slurm)
  - [Defining Reproducible Tasks](#defining-reproducible-tasks)
  - [Using Pixi on VSCode for Python Projects](#using-pixi-on-vscode-for-python-projects)
  - [Running RStudio With Pixi](#running-rstudio-with-pixi)
    - [Adding RStudio Task](#adding-rstudio-task)
    - [Interative Installation](#interactive-installation)
    - [Task Installation](#task-installation)
  - [Real Project Example](#real-project-example)

**Definitions:**

| Term | Definition |
|------|------------|
|**`pixi.toml`:**|Human-editable definition of dependencies and tasks.|
|**`pixi.lock`:**|Frozen dependency versions ensuring bit-for-bit reproducibility.|


# SOP

Every project should begin as usual (creating a github repo, cloning the repo to its respective location.) For the purposes of this SOP we will call this `260101_Smith_RNASeq`

## Initializing the Pixi Environment

To initialize a pixi environment you can run 1 of the following commands:

1. `pixi init`: Creates a `pixi.toml` file *directly* in your project root. 

2. `pixi init env_name`: Creates a folder in your project root named as env_name where all pixi information (i.e., toml, lock, and other information) will live. (*recommended*)

*note:* if option 2 is used, be sure to `cd` into the folder created by pixi to modify toml or add packages.

Following our example:

```
cd 260101_Smith_RNASeq
pixi init alignment
cd alignment
```

## Adding Software to the Environment 

Tools and software can be added to the Pixi environment:

**A.** directly from the command line with `pixi add` (which will automatically update the `toml` and `lock`)

**B.** Directly modifying the `toml` and running `pixi install`

In general, Pixi will try and first resolve packages through conda-based libraries (conda-forge, bioconda, etc.) before turning to PyPI, falling back on `UV`, a Rust-based package solver. 

Example of Option A:

```
pixi add samtools
```

Example of Option B (from a text editor such as nano or bbedit).
The fields you will mostly edit are:

-  **channels**:  Controls the channels packages are pulled from. Currently, pixi supports: conda-forge, bioconda, robostack, nvidia, pytorch. 

- **dependencies**: Add software dependencies here, OR have them automatically updated if you install interactively. 

- **tasks:** define reproducible tasks (more on this below.)

> [!IMPORTANT]
> **Pixi has a dedicated `pypi-dependencies table to manage packages from PyPI**

```
## This is how the toml file will look

authors = ["mikemartinez99 <mike.j.martinez99@gmail.com>"]
channels = ["conda-forge"]          #! Add channels here
name = "alignment"
platforms = ["osx-arm64"]
version = "0.1.0"

[tasks]

[dependencies]
samtools="*"    #! Add your tool and a version. "*" means find any!

```

Now install the packages

```
pixi install
```

If you want to add a package only available through pip (PyPI) you can use this command:

```
pixi add --pypi <somePackage>
```

You can also install python packages directly from github using this command:

```
pixi add --pypi "pixi add --pypi "mypkg @ git+https://github.com/myuser/myrepo.git@dev""
```

> [!IMPORTANT]
> **Installing R packages it a little more involved if the package is not hosted on CRAN. For more information on this, see the [R installation instructions](#running-rstudio-with-pixi)**

## Activating the Environments 

There are different ways to interact with the environment. 

#### Project Local Shell

To activate a project-local shell (similar to `conda activate`), run the following command from within the pixi folder:

```
# From within the alignment folder in our project directory...
pixi shell
```

#### Execute Commands Locally

Commands and scripts can be ran directly in the environment (similar to `singularity exec`).

```
# From any folder, as long as pixi shell has been activated
pixi run ../code/alignReads.sh
```

#### Using Pixi Environments with SLURM

sbatch scripts can be used in the same way, however rather than sourcing and activating the environment how you would with conda, you replace your script call with the `pixi run` command. Below is an example SBATCH script.

```
#!/bin/bash
#SBATCH --job-name=aln
#SBATCH --output=aln.out
#SBATCH --error=aln.err
#SBATCH --cpus-per-task=4
#SBATCH --mem=100G
#SBATCH --time=24:00:00
#SBATCH --partition=preempt1
#SBATCH --account=dac
#SBATCH --mail-user=f007qps@dartmouth.edu
#SBATCH --mail-type=FAIL

#-------------------------
# Run Alignment
#-------------------------
set -euo pipefail

cd /dartfs-hpc/rc/lab/G/GMBSR_bioinfo/Labs/smith/260101_RNASeq/alignment/

pixi run ../code/alignReads.sh

```

## Defining Reproducible Tasks

Reproducible tasks can be defined in the `toml` under the `tasks` section. These can be script calls, installations (*Note*, this will come in handy for R package installation), and more.

```
# TOML File

authors = ["mikemartinez99 <mike.j.martinez99@gmail.com>"]
channels = ["conda-forge", "bioconda"]
name = "alignment"
platforms = ["osx-arm64"]
version = "0.1.0"

[tasks]
qc = "python ../code/qc.py"

[dependencies]
samtools="*"
fastqc="*"
multiqc="*"    
```

Then we can run the task:

```
pixi run qc
```

## Using Pixi on VSCode for Python Projects

If using Pixi on VSCode for interactive python coding, take the following steps:

1. Install ipykernel in your pixi environment

```
pixi add ipykernel
```

2. Run this command after activating the shell 

```
pixi shell
python -m ipykernel install --user --name=alignment --display-name "Pixi:
aligment"
```

(**Note**, this command can be put under tasks in the `toml` to act as an alias of sorts) (example below)

```
authors = ["mikemartinez99 <mike.j.martinez99@gmail.com>"]
channels = ["conda-forge", "bioconda"]
name = "alignment"
platforms = ["osx-arm64"]
version = "0.1.0"

[tasks]
qc = "python ../code/qc.py"
kernel = "python -m ipykernel install --user --name=alignment --display-name 'Pixi: aligment'"

[dependencies]
samtools="*"
fastqc="*"
multiqc="*"    
```

Then we can run the task:

```
pixi run kernel
```

3. Run your python code interactively by specifying run blocks in your .py file using `#%%`. Hit `run block`. This should open an interactive window. In the top right hand corner, click "select kernel" --> Jupyter Kernels --> refresh (if needed) and select your kernel. 

You can also install packages through notebook blocks using Jupyters "!" syntax:

```
!pixi add numpy pandas matplotlib
```

## Running RStudio With Pixi

> [!IMPORTANT]
> **Running RStudio through pixi is a bit more complicated. For starters, if you are running RStudio locally while referencing project paths from GMBSR as it is mounted (eg., from `/Volumes/`) your pixi environment **needs to be made locally, not on a Dartmouth HPC**. For this, I recommend having a folder somewhere on your mac called where project environments will be stored. This allows you to run RStudio locally and still document your project. Thank's to Pixi's multi-platform support, the environment can be replicated across machines and you can still use the networked folders on Discovery localled by mounting.**


#### Adding RStudio Task

To run RStudio through a pixi environment, add the following task to your `toml`

```
authors = ["mikemartinez99 <mike.j.martinez99@gmail.com>"]
channels = ["conda-forge", "bioconda"]
name = "alignment"
platforms = ["osx-arm64", "linux-64"]
version = "0.1.0"

[tasks]
qc = "python ../code/qc.py"

kernel = "python -m ipykernel install --user --name=alignment --display-name 'Pixi: aligment'"

rstudio = "open -n -a RStudio"

[dependencies]
samtools="*"
fastqc="*"
multiqc="*"    
r-Seurat="4.4.0"
r-ggplot2="*"
```

Then you can run the task to open a new RStudio window:

```
pixi run rstudio
```

Any packages installed within your session will be added to your pixi folder's R library and will be accessible in future sessions, but **will not** be explicitly added to the `toml` or lock-file. To reflect R packages in these files, you need to install directly through pixi or through the `toml`. However, installing R packages on pixi is simple *if* the package is hosted on CRAN (i.e. Seurat.) To install a cran package, one can simply do:

```
pixi add r-Seurat
```

However, R packages hosted on Github or Bioconductor are more challenging. There are a few options for how to go about installing these. 

#### Interative Installation

From within the RStudio window opened by running your `pixi run rstudio` task, you can install packages as you normally would:

```R
if (!require("BiocManager", quietly = TRUE))
    install.packages("BiocManager")

BiocManager::install("GenomicRanges")
```

This will make `GenomicRanges` present in your `.libPaths` which should look something like this:

`/Volumes/GMBSR_bioinfo/labs/smith/260101_RNASeq/alignment/.pixi/envs/default/lib/R/library`

However, this will not add `GenomicRanges` to your `toml` file. It will be useable in all sessions, it just will not be reflected in the file.

 #### Task Installation

Task installation is helpful if you know exactly what packages you need and want to install everything in one shot. 

You can add a task like this to your `toml` and execute:

```
install-bioc = """
Rscript -e '
if (!requireNamespace("BiocManager", quietly=TRUE))
    install.packages("BiocManager", repos = "https://cloud.r-project.org");
BiocManager::install(c("DESeq2", "GenomicRanges"), ask=FALSE)
'
"""

```

Then run with `pixi run install-bioc`

> [!IMPORTANT]
> **If you set an installation task for bioconductor packages, it is imperative that you set `ask=FALSE` as bioconductor installations often prompt users for input regarding package version updates. If you do not do this, your script will time-out waiting for user input. The same is true for `repos = ...`**

> [!WARNING]
> **Pixi currently does not support bioconductor natively. Some bioconductor packages will work by either installing interactively or through a task, but not all are guarenteed to work (i.e., some bioconductor packages require dependencies that do not have binaries supported by pixi, so installations inherently fail). If this is the case, conda, or renv is a better alternative**


## Real Project Example

Below is an example for a basic differential expression project. After cloning the repo to our lab folder, I created a .Rproj file in the root. This was the process:

1. Environment initialization from a **local** folder on your machine. Here, my folder lives in `/Users/mike/Desktop/GDSC/`

```
cd /Users/mike/Desktop/GDSC/malaney/251201-Malaney-Lymphoma-Signature/
mkdir envs && cd envs
pixi init envs
```

2. Modification of `pixi.toml`

Key additions include the following:

- Under `workspace`, in the `platforms` filed, I Added both "osx=arm64" to facilitate functionality on my local machine when GMBSR is mounted, and `linux-64` to allow the builds on the HPC backend.

- Under `workspace` in the `channels` field, I added bioconda to facilitate installation of bioconda packages

- Under `tasks` I added `rstudio` which calls and activates my Rproj

- Under `tasks` I added `install-bioc` which calls for the installation of bioconductor packages needed for this project.

- Under `dependencies` I added all CRAN packages needed for this project.

```
[workspace]
authors = ["Mike Martinez <mike.j.martinez99@gmail.com>"]
channels = ["conda-forge", "bioconda"]
name = "envs"
platforms = ["osx-arm64", "linux-64"]
version = "0.1.0"

[tasks]

# Run RStudio project
rstudio = "open -a -n Rstudio /Volumes/GMBSR_bioinfo/Labs/malaney/251201-Malaney-Lymphoma-Signature/251201-Malaney-Lymphoma-Signature.Rproj"

# Install bioconductor packages
install-bioc = """
Rscript -e '
if (!requireNamespace("BiocManager", quietly=TRUE))
    install.packages("BiocManager", repos = "https://cloud.r-project.org");
BiocManager::install(c("DESeq2", "apeglm", "annotationDbi", "org.Hs.eg.db", "clusterProfiler"), ask = FALSE)
'
"""

# Install github packages
install-git = """
Rscript -e '
devtools::install_github("mikemartinez99/RGenEDA")
'
"""

[dependencies]
r-base = "4.4.1"
r-sessioninfo = "*"
r-remotes = "*"
r-devtools = "*"
r-tidyverse = "*"
r-dplyr = "*"
r-tidyr = "*"
"r-data.table" = "*"
r-pheatmap = "*"
r-ggplot2 = "*"
r-ggpubr = "*"
r-ggrepel = "*"
r-RColorBrewer = "*"
r-ggh4x = "*"

```

3. Install the environment

```
cd envs
pixi install
```

4. Run the `install-bioc` task

While the `pixi install` command is fast, this install will most likely take some time as it is installing through R and not pixi. 

```
pixi run install-bio
```

5. Run the `install-git` task

```
pixi run install-git
```






