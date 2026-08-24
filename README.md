# RiceOpticks Container — Usage Guide

A ready-to-run Apptainer image of **RiceOpticks** (GPU optical-photon
simulation, a fork of Opticks) built against **NVIDIA OptiX 9.0.0** and
**CUDA 13.0**, with Geant4 / CLHEP / Xerces-C supplied from the DUNE Spack
stack on CVMFS.

---

## What's inside

| Component      | Version                                    |
|----------------|--------------------------------------------|
| RiceOpticks    | v1.0r (commit `754d992`)                   |
| NVIDIA OptiX   | 9.0.0                                      |
| CUDA toolkit   | 13.0                                       |
| Geant4         | 11.2.2 (from CVMFS Spack `v1.2.2`)         |
| CLHEP          | 2.4.7.1 (from CVMFS)                       |
| Xerces-C       | 3.3.0 (from CVMFS)                         |
| Compiler       | gcc 12.5.0 (from CVMFS)                    |
| Base OS        | Rocky Linux 9 (EL9)                        |

This image contains only compiled binaries — the OptiX SDK headers are **not**
included. The OptiX runtime is provided by your host's NVIDIA driver.

---

## System requirements

This container is **not** self-contained. The host server must provide three
things, or the image will not run:

1. **CVMFS**, with these repositories mounted:
   - `/cvmfs/dune.opensciencegrid.org`
   - `/cvmfs/larsoft.opensciencegrid.org`

   Geant4, CLHEP, Xerces-C, and the matching gcc runtime load from CVMFS at
   run time. A server without CVMFS cannot run this image.

2. **NVIDIA driver R580 or newer**, and an **NVIDIA GPU of Turing
   architecture (compute capability 7.5) or newer** — e.g. RTX 20-series or
   later, or a Turing/Ampere/Ada/Hopper datacenter card. (CUDA 13 dropped
   support for older GPUs.)

3. **Apptainer** (or Singularity) installed.

Check all three before running anything:

```bash
nvidia-smi                                               # driver >= 580, a Turing+ GPU
ls /cvmfs/dune.opensciencegrid.org /cvmfs/larsoft.opensciencegrid.org
apptainer --version
```

If `nvidia-smi` reports a driver below 580, or `/cvmfs` is missing, the image
will not work on that server — that is a host provisioning issue, not a
container problem.

---
