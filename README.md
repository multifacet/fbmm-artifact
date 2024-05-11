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
- `python3` with `matplotlib`, `numpy`, `scipy`, `pandas`, and `seaborn`

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
    git submodule update --init --recursive -- fbmm-workspace
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
    ./target/debug/expjobserver --allow_snap_fail ../fbmm-workspace/runner/target/debug/runner ~/fbmm_logs example.log.yml
    ```
    This will start an instance of the jobserver.
    `~/fbmm_logs` is the directory the jobserver will save the log output of the experiments we run.
    `~/fbmm_results` is the directory the jobserver will save the experimental results to.

6. Setup the _test_ machine(s)
    ```sh
    cd ./fbmm-artifact/jobserver/
    ssh -p <ssh port> <user>@<test url>
    exit
    ssh -p <ssh port> <user>@<test ip>
    exit
    ./target/debug/j machine setup -m <test url>:<ssh port> -c fbmm "setup_wkspc {MACHINE} <user> --clone_wkspc --wkspc_branch atc-artifact --host_bmks --host_dep --unstable_device_names --resize_root --spec_2017 <spec path>" "setup_kernel {MACHINE} <user> --branch atc-artifact --repo github.com/multifacet/fbmm --install_perf --build_mmfs +CONFIG_TRANSPARENT_HUGEPAGE -CONFIG_PAGE_TABLE_ISOLATION -CONFIG_RETPOLINE +CONFIG_GDB_SCRIPTS +CONFIG_FRAME_POINTERS +CONFIG_IKHEADERS +CONFIG_SLAB_FREELIST_RANDOM +CONFIG_SHUFFLE_PAGE_ALLOCATOR +CONFIG_FS_DAX +CONFIG_DAX +CONFIG_BLK_DEV_RAM +CONFIG_FILE_BASED_MM +CONFIG_BLK_DEV_PMEM +CONFIG_ND_BLK +CONFIG_BTT +CONFIG_NVDIMM_PFN +CONFIG_NVDIMM_DAX +CONFIG_X86_PMEM_LEGACY -CONFIG_INIT_ON_ALLOC_DEFAULT_ON"
    ```
    Where
    - `test url` is the URL of the _test_ machine being setup
    - `test ip` is the IP address of the _test_ machine being setup
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
./target/debug/j job add fbmm "fbmm_exp {MACHINE} <user> --disable_thp --numactl --fbmm --basicmmfs 16777216 alloctest 1 100000 --threads 1" ~/fbmm_results
```

After a few minutes, the job should complete successfully, which can be seen by checking its status with
```sh
./target/debug/j job ls
```

## FBMM Translation Layer Overhead
This section will describe the experiments used to measure the performance of the FBMM Translation Layer, which is discussed in Section 4 and Table 3 of the paper.

### Running the Experiments
The following command will run the experiments used to generate Table 3.
```sh
./target/debug/j job matrix add -x 10 --max_failures 1 fbmm "fbmm_exp {MACHINE} <user> --disable_thp --numactl {FBMM} alloctest {SIZE} 100000 --threads 1 {POPULATE}" ~/fbmm_results \
    FBMM=,"--fbmm --basicmmfs 16777216" \
    SIZE=1,2,8,32,128 \
    POPULATE=,"--populate"
```
This command tells the jobserver to create a "matrix" of jobs, where it runs jobs with every combination of arguments provided at the end of the command.
The `-x` flag at the beginning is the number of times each combination should be run.
The `--max_failures` flag limits the number of times a job in the matrix can be retried before aborting.
The `FBMM` argument is used to choose between running base linux or FBMM with the BasicMFS.
The `SIZE` argument specifies the size in pages of each allocation the benchmark makes.
The `POPULATE` argument specifies whether or not the allocations should be backed by physical memory.

### Collecting the Results
The following command will parse the output of the experiments and put them into a CSV file.

```sh
./target/debug/j job stat --csv --only_done --results_path "" --jid --cmd \
    --id <matrix id> --mapper ../fbmm-workspace/scripts/extract-alloctest.py \
    > table3.csv
```
where `matrix id` is the job id of the job matrix.
This is printed by `j` when the matrix is started, and can be found by running
```sh
./target/debug j job ls
```

The "Kernel," "Alloc Size," and "Populate" columns in the CSV file uniquely identify the experiment configuration.
The "Map Time," and "Unmap Time" columns are the measured time it took in CPU cycles to map/unmap all of the allocations.
To get an average of the measurements relevant to these experiments, take the mean of the "Map Time" and "Unmap Time" within each experiment configuration.
The script orders the columns in alphabetical order, so it might be easier to rearrange them in a spreadsheet software before continuing.

CPU cycles can be converted to microseconds, like reported in the paper, with the following equation:

$$ microseconds = (\frac{x}{100000} / freq) * 1000000 $$

where $x$ is the average map/unmap cycles, and $freq$ is the CPU frequency when the CPU scaling governor is set to "performance."
On a c220g1 machine, that is $3200000000 Hz$.

## TieredMFS Evaluation
This section describes how to run the experiments used to generate Figure 3 and Table 5 of the paper.

### TieredMFS GUPS Experiment
The following command will run the experiments used to generate Table 5
```sh
./target/debug/j job matrix add -x 5 --max_failures 1 fbmm "fbmm_exp {MACHINE} <user> --disable_thp {EXP} gups --move_hot 35 33" \
    ~/fbmm_results \
    EXP="--numactl","--dram_size 68 --dram_start 12","--fbmm --tieredmmfs --dram_size 8 --dram_start 4 --pmem_size 45 --pmem_start 68"
```

After all of those experiments finish, this command will parse and collect the results into a CSV file
```sh
./target/debug/j job stat --csv --only_done --results_path "gups" --jid --cmd --class \
    --id <matrix id> --mapper ../fbmm-workspace/scripts/extract-gups.py \
    > table5.csv
```

The "Type" column of the CSV will uniquely identify the experiment configuration.
The "GUPS" column is the measured GUPS performance of that run.
After averaging the "GUPS" column for each "Type," the result reported in Table 5 are the Linux Split and FBMM types divided by the GUPS value of the Linux Local type.

### TieredMFS Memcached
The following command will run the experiments used to generate Figure 3
```sh
./target/debug/j job matrix add -x 50 --max_failures 1 fbmm "fbmm_exp {MACHINE} <user> --disable_thp {EXP} memcached --op_count 10000000 --read_prop 1.0 --update_prop 0.0 40 " \
    ~/fbmm_results \
    EXP="--numactl","--dram_size 64 --dram_start 14","--fbmm --tieredmmfs --dram_size 10 --dram_start 4 --pmem_size 45 --pmem_start 68"
```
Memcached experiments run for a while, so for the sake of time, I would recommended adding multiple machines to the jobserver using the step 6 of the Setup section, and running fewer than 50 runs per experiment configuration (maybe 10-25 depending on how many machines are used).

The results are parsed using the following command
```sh
./target/debug/j job stat --csv --only_done --results_path "ycsb" --jid --cmd --class \
    --id <matrix id> --mapper ../fbmm-workspace/scripts/extract-ycsb.py \
    > figure3.csv
```

Then, to generate the figure, run
```sh
~/fbmm-artifact/fbmm-workspace/scripts/plot-ycsb-box.py figure3.csv
```

## BWMFS Evaluation
The following command will run the experiments used to generate Figure 4
```sh
./target/debug/j job matrix add -x 5 --max_failures 1 fbmm "fbmm_exp {MACHINE} <user> --disable_thp --numactl {EXP} stream --threads 8" \
    ~/fbmm_results \
    EXP=,"--fbmm --bwmmfs --node_weight 0:1 --node_weight 1:1","--fbmm --bwmmfs --node_weight 0:2 --node_weight 1:1","--fbmm --bwmmfs --node_weight 0:3 --node_weight 1:1","--fbmm --bwmmfs --node_weight 0:3 --node_weight 1:2","--fbmm --bwmmfs --node_weight 0:5 --node_weight 1:2","--fbmm --bwmmfs --node_weight 0:1 --node_weight 1:2","--fbmm --bwmmfs --node_weight 0:1 --node_weight 1:3","--fbmm --bwmmfs --node_weight 0:2 --node_weight 1:3"
```

The results are parsed using the following command
```sh
./target/debug/j job stat --csv --only_done --results_path "stream" --jid --cmd \
    --id <matrix id> --mapper ../fbmm-workspace/scripts/extract-stream.py \
    > figure4.csv
```

Then, to generate the figure, run
```sh
~/fbmm-artifact/fbmm-workspace/scripts/plot-stream-results.py figure4.csv
```

## ContigMFS Evaluation

The following commnad will run the experiments used to generate Table 6
```sh
./target/debug/j job matrix add -x 1 --max_failures 1 fbmm "fbmm_exp {MACHINE} <user> --disable_thp --badger_trap --fbmm --contigmmfs {WKLD}" \
    ~/fbmm_results \
    WKLD="spec17 mcf","spec17 cactubssn","gups --move_hot 35 33"
```

The we only have one results per workload, so we don't have a fancy parsing script for these experiments like the others.
To find the output files for these experiments, run
```sh
./target/debug/j job stat --text --only_done --results_path "badger_trap" --cmd --jid --id <matrix id>
```

If you open the output files, you will see some statistics captured by the badger trap utility.
The measurements in Table 6 are calculated by dividing the "Range TLB hit detected" count by the "DTLB miss detected" count.
