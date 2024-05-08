# Artifact for FBMM (ATC'24)
TODO: Add link to artifact github repo
TODO: Move the submodule repos to the multifacet github

## Overview
This artifact contains:

- `README.md`: This README
- `paper.pdf`: The version of this paper originally submitted to ATC. Note, for anonymity reasons, we renamed "File Base Memory Management (FBMM)" to "Extensible Memory Management (EMM)" and "Memory Management File System (MFS)" to "File System Memory Manager (FSMM)" in this draft.
- `fbmm/`: A git submodule containing the Linux Kernel with FBMM, written on top of Linux v6.2. It also contains the source code for the MFSs described in the paper.
- `fbmm-workspace`: A git submodule containing the `runner`, our tool to run experiments, as well as benchmarks used, and scripts to aggregate experiment data.
- `jobserver/`: A git submodule containing the tool we use to orchastrate the experiments we run

Because our artifact is an entire kernel, our evaluation uses two classes of machines: one to drive the experiments (the _driver_ machine) and one or more where the experiments are run (the _test_ machines).
This allows the experiments to be run on a bare metal machine, without any interference from virtualization, and to make automation easier.

The _test_ machines must have the FBMM kernel installed with Ubuntu 20.04, while the _driver_ machine can be any recent Linux machine with [Rust installed](https://www.rust-lang.org/tools/install).

## Artifact Claims
Running the experiments described in this document on the same hardware used to evaluate FBMM in our system (e.g. Cloudlab c220g1, Ubuntu 20.04) will yield comparable results to those described in the paper.

Specifically, the claims about
- FBMM Translation Layer overhead (Table 3)
- TieredMFS outperforms base Linux (Figure 3, Table 5)
- BWMFS can be used to expand the bandwidth capacity of applications (Figure 4)
- ContigMFS reduces the number of TLB misses an application suffers (Table 6)

## System requriements

_Driver_ machine:

- A recent Linux distro with standard utilities. We mostly used Ubuntu 20.04/22.04
- [Rust](https://www.rust-lang.org/tools/install) to compile the `jobserver` and `runner`
- _Passwordless_ SSH access to the driver machines, and a stable network connection that allows for hours-long SSH sessions
- A SPEC2017 benchmark ISO image. An image is not included in this artifact for licensing reasons.
- `python3` with `matplotlib`, `numpy`, and `scipy`

_Test_ machine:
- At least two NUMA nodes
- Ubuntu 20.04
- We recommend the Cloudlab c220g1 machines to most closely recreate the results from the paper.

## Setup
All of the following commands will be run on the **_driver_** machine.

1. Clone this repo
    ```sh
    git clone https://github.com/multifacet/fbmm-artifact
    ```

2. Initialize the submodules
    ```sh
    cd fbmm-artifact
    git submodule update --init -- fbmm-workspace
    git submodule update --init -- jobserver
    ```

3. Build the jobserver
    ```sh
    cd jobserver
    cargo build
    cd ../
    ```

4. Build the runner
    ```sh
    cd fbmm-workspace/runner/
    cargo build
    cd ../../
    ```

5. Start the jobserver
    In a different terminal, run
    ```sh
    mkdir -p ~/fbmm_logs
    mkdir -p ~/fbmm_results
    cd ./fbmm-artifact/jobserver/
    ./target/debug/expjobserver ../fbmm-workspace/runner/target/debug/runner ~/fbmm_logs example.log.yml
    ```
    This will start an instance of the jobserver.
    `~/fbmm_logs` is the directory the jobserver will save the log output of the experiments we run.
    `~/fbmm_results` is the directory the jobserver will save the experimental results to.

6. Setup the _test_ machine(s)
    ```sh
    cd ./fbmm-artifact/jobserver/
    ssh -p <ssh port> <_test_ url>
    exit
    ssh -p <ssh port> <_test_ ip>
    exit
    ./target/debug/j machine setup -m <_test_ url>:<ssh port> -c fbmm "setup_wkspc {MACHINE} <user> --clone_wkspc --host_bmks --host_dep --unstable_device_names --resize_root --spec_2017 <spec path>" "setup_kernel {MACHINE} {USER} --branch atc-artifact --repo github.com/multifacet/fbmm --install_perf --build_mmfs +CONFIG_TRANSPARENT_HUGEPAGE -CONFIG_PAGE_TABLE_ISOLATION -CONFIG_RETPOLINE +CONFIG_GDB_SCRIPTS +CONFIG_FRAME_POINTERS +CONFIG_IKHEADERS +CONFIG_SLAB_FREELIST_RANDOM +CONFIG_SHUFFLE_PAGE_ALLOCATOR +CONFIG_FS_DAX +CONFIG_DAX +CONFIG_BLK_DEV_RAM +CONFIG_FILE_BASED_MM +CONFIG_BLK_DEV_PMEM +CONFIG_ND_BLK +CONFIG_BTT +CONFIG_NVDIMM_PFN +CONFIG_NVDIMM_DAX +CONFIG_X86_PMEM_LEGACY -CONFIG_INIT_ON_ALLOC_DEFAULT_ON"
    ```
    Where
    - `_test_ url` is the URL of the machine being setup
    - `_test_ ip` is the IP address of the machine being setup
    - `ssh port` is the SSH port to use for that machine. Usually 22
    - `user` is the username to SSH into the _test_ machine with
    - `spec path` is the path to the SPEC2017 ISO

    We need to ssh into the machine before running the setup scripts so it is in the known_hosts file.

    `./target/debug/j` is the client program that talks to the jobserver to add new jobs.
    To see the list of jobs, run
    ```sh
    ./target/debug/j job ls
    ```
    Which will show the list of jobs, their current status (e.g., running, done), and their unique job id (jid).

    To watch the progress of a running command, run
    ```sh
    tail -f ~/fbmm_logs/<jid>-*
    ```

    If there is a machine you no longer want to use for experiments, run
    ```sh
    ./target/debug/j machine rm -m <_test_ url>:<ssh port>
    ```

## Kick The Tires
To test if everything is up and running, run the following command to run an experiment where several kernel allocations are made using FBMM
```sh
cd ./fbmm-artifact/jobserver/
./target/debug/j job add fbmm "fbmm_exp {MACHINE} <user> --disable_thp --numactl --fbmm --basicmmfs 16777216 alloctest 1 100000 --threads 1"
```

After a few minutes, the job should complete successfully, which can be seen by checking its status with
```sh
./target/debug/j job ls
```
