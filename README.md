# Artifact for FBMM (ATC'24)
TODO: Add link to artifact github repo

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



