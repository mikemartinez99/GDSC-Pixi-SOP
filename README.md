# Dartmouth Genomic Data Science Core - Pixi SOP

**Author: Mike Martinez M.S.**

**Date: December 12, 2025**

**Purpose:** 

To provide a standardized framework for implmenting and managing software environments for core projects using Pixi

**Scope:** 

This SOP applies to any projects where a reproducible environment is needed, including R, Python, command-line tools, and mixed-language workflows. Pixi may not be applicable for every project due to current limitations in support for Bioconductor tools (R) as of December 2025. 

**Responsibilities:** 

Initialization and management of environment should be done by the project owner. Collaborators within the core should be able to edit as needed. Clients receiving environments should receive the `pixi.toml` and `pixi.lock`.

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

#### Adding RStudio Task

To run RStudio through a pixi environment, add the following task to your `toml`

```
authors = ["mikemartinez99 <mike.j.martinez99@gmail.com>"]
channels = ["conda-forge", "bioconda"]
name = "alignment"
platforms = ["osx-arm64"]
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
    install.packages("BiocManager");
BiocManager::install(c("DESeq2", "GenomicRanges"), ask=FALSE)
'
"""

```

Then run with `pixi run install-bioc`

> [!IMPORTANT]
> **If you set an installation task for bioconductor packages, it is imperative that you set `ask=FALSE` as bioconductor installations often prompt users for input regarding package version updates. If you do not do this, your script will time-out waiting for user input**

> [!WARNING]
> **Pixi currently does not support bioconductor natively. Some bioconductor packages will work by either installing interactively or through a task, but not all are guarenteed to work (i.e., some bioconductor packages require dependencies that do not have binaries supported by pixi, so installations inherently fail). If this is the case, conda, or renv is a better alternative**






