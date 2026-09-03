# NYBG HPC PRACTICAL GUIDE: CLUSTER ARCHITECTURE, SOFTWARE ENVIRONMENT AND JOB SUBMISSION

By Kelly T. Bocanegra-González September 2026

This section documents how the NYBG HPC cluster is organised, how computational resources are managed, how software is accessed through modules, and how persistent shell configuration differs from project-specific software environments.

It is intended as a practical reference to make routine use of the cluster easier for NYBG users, particularly when setting up software environments, accessing computational resources, and running bioinformatics workflows in a consistent and reproducible way.

# 1. CLUSTER ARCHITECTURE

The NYBG HPC cluster is a shared computing system composed of a login environment, compute nodes, storage areas, and a workload manager.

The login or head node is the point of entry to the cluster. It is used for tasks such as navigating directories, organising files, editing scripts, transferring data, inspecting available software, and submitting computational jobs.

Computationally intensive analyses should generally not be run directly on the login node. Instead, they are submitted to compute nodes through the SLURM workload manager.

## 1.1. Login

The main entry point for institutional resources is: https://nybg.sharepoint.com/ From there, users can access internal documentation, credentials-related information, and other NYBG resources associated with their account. For example my access: ```ssh <username>@<cluster-address>``` 

## 1.2. Compute nodes

General information about the available partitions and their status can be obtained with: ```bash sinfo```

To inspect individual nodes in more detail: ```bash sinfo -N -l```

To display node name, number of CPUs, available memory, and node state: ```bash sinfo -N -o "%N %c %m %t"```

To inspect the configuration of the default partition: ```bash scontrol show partition defq```

Conceptually, the cluster can be represented as:

```
HPC cluster
│
├── Login / head node
│   ├── file management
│   ├── script editing
│   ├── data transfer
│   └── job submission
│
├── SLURM workload manager
│
├── Compute nodes
│   └── computational analyses
│
└── Storage
    ├── home directory
    └── scratch / project space
```

## 1.3. SLURM

SLURM manages the allocation of computational resources, including CPUs, memory, nodes, partitions, and running jobs. The SLURM module is loaded with:

```bash module add slurm/slurm/25.05```

To inspect the jobs associated with the current user: ```bash squeue -u $USER```

## 1.4. Shell environment: ~/.bashrc

The file: ```bash ~/.bashrc ``` is a Bash shell configuration file associated with the user's account. It is fundamentally different from a Conda environment. A Conda environment isolates software for a particular workflow, whereas `.bashrc` defines settings and commands that should be available as part of the user's **general shell execution environment**.

### 1.4.1. Location and behaviour of `~/.bashrc`

The symbol: ```bash ~``` represents the user's home directory. Therefore: ```bash ~/.bashrc``` refers to the `.bashrc` file located in the user's home directory, regardless of the current working directory. For example, it remains the same file whether the user is currently working in the home directory, a project directory, or scratch space.

When a new interactive Bash shell starts, the commands contained in `.bashrc` are executed automatically. This makes `.bashrc` useful for settings and infrastructure that should normally be available every time a new cluster session is opened. For this cluster, the following modules are useful as part of the general execution environment:

```bash
module add slurm/slurm/25.05
module load shared
module load miniconda/26.5.3-2
```

Adding these commands to `.bashrc` means that they do not need to be loaded manually every time a new shell session is started. After modifying `.bashrc`, the configuration can be applied immediately with: ```bash source ~/.bashrc```

`source ~/.bashrc` re-executes the commands contained in `.bashrc` in the current shell. It therefore allows changes to take effect without logging out and starting a new session.

## 1.6. Software access through modules

Software installed centrally on the cluster is commonly accessed through the **Lmod module system**. A module does not normally install a new copy of the software into the user's directory. Instead, loading a module modifies the current shell environment so that the corresponding program and its dependencies become accessible.

For example: ```bash module load shared``` loads a general software collection or prerequisite layer.

Some programs are only accessible after another module has first been loaded.

### 1.6.1. Searching for software

To search the full module tree: ```bash module spider <software-name>```

For example: ```bash module spider lftp``` or: ```bash module spider conda ```

`module spider` is particularly useful because it can identify software that is not directly visible with `module avail` and can show which prerequisite modules must be loaded first.

### 1.6.2. Example: Accessing software for raw-data download

As an example, the sequencing provider recommended using `lftp` to download the raw sequencing data.

`lftp` is a command-line file-transfer tool designed to transfer files between local and remote systems. It supports protocols such as FTP, FTPS, HTTP, and SFTP and includes features that are particularly useful when transferring large sequencing datasets, including:

* recursive directory mirroring;
* continuation of interrupted transfers;
* automatic retries;
* parallel file transfer.

These features make it particularly useful for downloading large raw sequencing datasets. 

In this case, `lftp` was not available as a cluster module: ```bash module spider lftp```

The search returned no available module.

However, Miniconda was available: ```bash module spider conda```

- Miniconda is a lightweight distribution of Conda that provides the conda package and environment manager without installing a large collection of preconfigured software. It is commonly used to create isolated software environments for different bioinformatics workflows. https://www.anaconda.com/docs/getting-started/concepts/anaconda-or-miniconda?

The cluster indicated that the `shared` module had to be loaded before Miniconda: 

```bash
module load shared
module load miniconda/26.5.3-2
```

The software hierarchy for this example is therefore:

```
Cluster
│
└── Lmod module system
    │
    └── shared
        │
        └── Miniconda
            │
            └── Conda environment
                │
                └── lftp
```

Here:
* `shared` is a cluster module that provides access to a collection of centrally installed software;
* `Miniconda` is the software used to create and manage Conda environments;
* the Conda environment is created and controlled by the user;
* `lftp` is an individual software tool installed inside that environment.

---

## 1.7 Software environments

A Conda environment is an isolated software environment containing a defined set of programs and their dependencies. The main purpose of an environment is to keep software required for one workflow separated from software required for another workflow. For example, an environment dedicated to downloading sequencing data can be created with:

```bash conda create -n download_tools -c conda-forge lftp``` and activated with: ```bash conda activate download_tools```

The structure can be conceptualised as:

```
Miniconda
│
├── download_tools (Environment 1)
│   └── lftp
│
├── Environment 2
│   └── assembly software
│
├── Environment 3
│   └── quality-control software
│
└── Environment 4
    └── analyses software
```

Different environments can contain different programs, dependencies, or versions of the same program without interfering with one another. This is particularly important in bioinformatics, where different analytical tools may depend on incompatible versions of Python, libraries, compilers, or other software. For this reason, Conda environments should generally be used for **workflow-specific or analysis-specific software**.

An environment can be activated when its tools are required:

```bash conda activate download_tools``` and deactivated when they are no longer needed: ```bash conda deactivate```

The environment remains stored in the user's Conda area after logging out of the cluster. It does not need to be recreated every time a new session is opened.

### 1.7.1. `.bashrc` vs Conda environment

The main difference is their **scope and purpose**. `.bashrc` prepares the general shell environment. A Conda environment provides an isolated collection of software for a particular workflow.

Conceptually:

```
User opens a cluster session
        │
        ▼
~/.bashrc
        │
        ├── SLURM
        ├── shared
        └── Miniconda
                │
                ▼
        Conda environments
                │
                ├── download_tools (Environment 1)
                ├── Environment 2
                ├── Environment 3
                └── Environment 4
```

Anything loaded through `.bashrc` becomes part of the general shell environment whenever a new session starts. A Conda environment, in contrast, can be activated only when required and keeps its software and dependencies separated from other environments.

### 1.7.2. What should go in `.bashrc`?

The `.bashrc` file should remain relatively minimal and contain settings or infrastructure that are broadly useful across cluster sessions.

Appropriate examples include:

* essential cluster infrastructure;
* general module prerequisites;
* access to Miniconda;
* permanent `PATH` modifications;
* useful shell aliases;
* broadly applicable environment variables.

For the current cluster setup, reasonable permanent entries are:

```bash
module add slurm/slurm/25.05
module load shared
module load miniconda/26.5.3-2
```

These commands prepare the general execution environment without determining which project-specific analytical software is active.

### 1.7.3. What should NOT normally go in `.bashrc`?

Project-specific software and Conda environments should generally **not** be activated automatically through `.bashrc`.

For example: ```bash conda activate download_tools``` should normally not be placed in `.bashrc`.

Instead, it should be activated manually when the tools contained in that environment are required.

Likewise, automatically loading many analytical programs such as:

```bash
module load samtools
module load bwa
module load blast
```

is generally not advisable unless those programs are genuinely needed in almost every shell session. Automatically loading project-specific software can:

* create dependency conflicts;
* load unnecessary software in every session;
* make it harder to identify which software environment was used for a particular analysis;
* cause problems when different projects require incompatible software versions;
* reduce workflow reproducibility.

A better approach is to keep `.bashrc` focused on general infrastructure and activate the appropriate software environment for each analytical stage.

For example: ```bash conda activate download_tools```

when downloading raw sequencing data, and later:

```bash conda activate genome_assembly```

# 1. JOB SUBMISSION (COMMING SOON)