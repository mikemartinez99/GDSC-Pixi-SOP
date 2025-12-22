# Dartmouth Genomic Data Science Core - Pixi SOP

**Author: Mike Martinez M.S.**

**Date: December 12, 2025**

**Purpose:** 
To provide a standardized framework for implmenting and managing software environments for core projects using Pixi, a versatile package manager that facilitates reproducibility across machines.

**Scope:** 
This SOP applies to any projects where a reproducible environment is needed, including R, Python, command-line tools, and mixed-language workflows. Pixi may not be applicable for every project due to current limitations in support for Bioconductor tools (R) as of December 2025 (see [Running RStudio with Pixi](#running-rstudio-with-pixi))

[**Pixi Documentation**](https://pixi.prefix.dev/v0.62.1/)

--------

# Table of Contents

- [Installations and prerequisites](#installations-and-prerequisites)
  - [Local installation](#local-installation)
  - [Remote configuration](#remote-configuration)
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
    - [Bioconda Recipies](#bioconda-recipies)
    - [Task Installation](#task-installation)
  - [Real Project Example in R](#real-project-example-in-r)
  - [Real Project Example in Python](#real-project-example-in-python)

**Definitions:**

| Term | Definition |
|------|------------|
|**`pixi.toml`**|Human-editable project configuration file that defines the workspace metadata, dependency requirements, supported platforms, and reproducible tasks. Serves as the single source of truth for the software environment.
|
|**`pixi.lock`**|Auto-generated lockfile that records the exact resolved package versions and build hashes for each platform, ensuring bit-for-bit reproducibility across systems and over time. Should be committed to version control.
|
|**workspace**| Section in pixi.toml that defines high-level project metadata, including project name, version, authors, Conda channels, and supported operating systems/platforms. Establishes the scope and identity of the project environment.
|
|**channels**|Package repositories. For our use cases, most packages will be through `conda-forge`, `bioconda`|
|**tasks**|Reproducible tasks that can be defined through the `toml`|
|**dependencies**|Section in pixi.toml that specifies required software packages and version constraints (e.g., Python, R, bioinformatics tools). Pixi resolves these into concrete installs via the lockfile.|


# Installations and prerequisites


> [!IMPORTANT]
> **Pixi is already installed on Discovery in GMBSR's `shared-software` folder, but needs to be added to `$PATH`. Pixi needs to be installed *locally* for RStudio projects**


## Local Installation


To install Pixi, run this command:

**(Linux/macOS):**

```
curl -fsSL https://pixi.sh/install.sh | sh
```

If your system does not use curl, you can use `wget` or `brew`

```
wget -qO- https://pixi.sh/install.sh | sh
```

```
brew install pixi
```

**Windows**

```
winget install prefix-dev.pixi
```

## Remote configuration

Pixi is already installed on Discovery in the `shared-software` folder. To access it, it needs to be added to your `bashrc` path. 

```
echo 'export PATH="/dartfs-hpc/rc/lab/G/GMBSR_bioinfo/misc/shared-
software:$PATH"' >> ~/.bashrc

source ~/.bashrc
```

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

- **channels**:  Controls the channels packages are pulled from. Currently, pixi supports: conda-forge, bioconda, robostack, nvidia, pytorch. 

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
> **Running RStudio through pixi is a bit more complicated. For starters, if you are running RStudio locally while referencing project paths from GMBSR as it is mounted (eg., from `/Volumes/`) your pixi environment *needs to be made locally, not on a Dartmouth HPC*. For this, I recommend having a folder somewhere on your mac called where project environments will be stored. This allows you to run RStudio locally and still document your project. Thank's to Pixi's multi-platform support, the environment can be replicated across machines and you can still use the networked folders on Discovery localled by mounting.**


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

R packages hosted on Github or Bioconductor can also be installed (with some caveats)

#### Bioconda Recipies

Bioconductor packages can be installed directly through the `toml` like a normal python package or CRAN package as long as they have a bioconda recipie. Bioconda needs to be specified as a channel. Bioconda recipies for R-bioconductor packages use the prefic `bioconductor-packagename> all lowercase. An example is below:

In this example, we install an R package from github, R packages from CRAN, and R packages from bioconductor (through bioconda)

```
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
r-MatchIt = "*"
bioconductor-deseq2="*"
bioconductor-apeglm="*"
bioconductor-clusterprofiler="*"    # Bioconda packages
bioconductor-annotationdbi="*"
"bioconductor-org.hs.eg.db"="*"

```

 #### Task Installation

Task installation is helpful for packages that are only available on github repositories. 

You can add a task like this to your `toml` and execute it using `pixi run taskname`:

```
[tasks]

# Run RStudio project
rstudio = "open -n -a Rstudio /Volumes/GMBSR_bioinfo/Labs/malaney/251201-Malaney-Lymphoma-Signature/251201-Malaney-Lymphoma-Signature.Rproj"

# Install github packages
install-git = """
Rscript -e '
devtools::install_github("mikemartinez99/RGenEDA")
'
"""

```

Then run with `pixi run install-git`

> [!IMPORTANT]
> **If you set an installation task for bioconductor packages, it is imperative that you set `ask=FALSE` as bioconductor installations often prompt users for input regarding package version updates. If you do not do this, your script will time-out waiting for user input. The same is true for `repos = ...`**

> [!WARNING]
> **Pixi currently does not support bioconductor natively. Some bioconductor packages will work by either installing interactively, or through a task, but not all are guarenteed to work. The best work around to have these packages installed and documented in the toml and lock is to find a bioconda recipie and install through the toml (see real world example below**


## Real Project Example in R

Below is an example for a basic differential expression project. After cloning the repo to our lab folder, I created a .Rproj file in the root. This was the process:

1. Environment initialization from a **local** folder on your machine. Here, my folder lives in `/Users/mike/Desktop/GDSC/`

```
cd /Users/mike/Desktop/GDSC/malaney/251201-Malaney-Lymphoma-Signature/
mkdir envs && cd envs
pixi init envs
```

2. Modification of `pixi.toml`

Key additions include the following:

- Under `workspace` in the `channels` field, I added `bioconda` to facilitate installation of bioconductor packages (R)

- Under `tasks` I added `rstudio` which calls and activates my Rproj in a local RStudio window (for when I mount the cluster)

- Under `tasks` I added `install-git` which calls for the installation of packages only available through github

- Under `dependencies` I added all CRAN, bioconda, and other packages needed for this project.

```
[workspace]
authors = ["Mike Martinez <mike.j.martinez99@gmail.com>"]
channels = ["conda-forge", "bioconda"]
name = "envs"
platforms = ["osx-arm64","linux-64"]
version = "0.1.0"

[tasks]

# Run RStudio project
rstudio = "open -n -a Rstudio /Volumes/GMBSR_bioinfo/Labs/malaney/251201-Malaney-Lymphoma-Signature/251201-Malaney-Lymphoma-Signature.Rproj"

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
r-MatchIt = "*"
bioconductor-deseq2="*"
bioconductor-apeglm="*"
bioconductor-clusterprofiler="*"
bioconductor-annotationdbi="*"
"bioconductor-org.hs.eg.db"="*"

```

3. Install the environment

```
cd envs
pixi install
```

4. Run the `install-git` task

```
pixi run install-git
```

5. Call the Rproj task to open the project in RStudio, with paths mounted to the cluster

```
pixi run rstudio
```

## Real Project Example in Python

1. Navigate to the project directory on Discovery/Andes/Polaris. Initialize a pixi directory and provide it a name.

```
pixi init multivelo
cd multivelo

```

2. Modify the `toml` by definig channels, tasks, and packages. 

> [!IMPORTANT]
> **If you want to run your python code interactively, you need to install `ipykernel` in the environment and include a task to activate the env as a kernel**

```
[workspace]
authors = ["Mike Martinez <f007qps@dartmouth.edu>"]
channels = ["conda-forge", "bioconda"]
name = "multivelo"
platforms = ["linux-64"]
version = "0.1.0"

[tasks]
kernel = "python -m ipykernel install --user --name multivelo-pixi --display-name 'Python (MultiVelo Pixi)'"

[dependencies]
python = "3.11.*"
pip = "*"
ipykernel = "*"
multivelo = "*"
loompy = ">=3.0.8,<4"
```

3. Activate the pixi environment

```
pixi shell
```

4. Run the `kernel` task to register the Jupyter kernel

```
pixi run kernel
```

5. In VS Code or Jupyter, select “Python (MultiVelo Pixi)” from the available kernel list.





