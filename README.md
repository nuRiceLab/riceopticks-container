# RiceOpticks Container — Usage Guide

A ready-to-run Apptainer image of **RiceOpticks** (GPU optical-photon
simulation, a fork of Opticks) built against **NVIDIA OptiX 9.0.0** and
**CUDA 13.0**, with Geant4 / CLHEP / Xerces-C / ROOT supplied from the DUNE
Spack stack on CVMFS.

The image serves both use cases: running Opticks executables, and compiling
your own Geant4 + Opticks application against it.

---

## Quick start

```bash
apptainer pull riceopticks.sif oras://ghcr.io/nuricelab/riceopticks:v1.0r
apptainer run --nv --bind /cvmfs riceopticks.sif bash
```

That drops you into a shell with the full environment already loaded. See
[Getting the image](#getting-the-image) for cache settings and checksums, and
[Running](#running) for why `run` rather than `shell`.

---

## What's inside

| Component      | Version                                        |
|----------------|------------------------------------------------|
| RiceOpticks    | v1.0r — commit `754d99225a60aa2370c1953064ee00375490eafc` |
| NVIDIA OptiX   | 9.0.0 headers (from [NVIDIA/optix-dev](https://github.com/NVIDIA/optix-dev) `v9.0.0`) |
| CUDA toolkit   | 13.0.3                                         |
| Geant4         | 11.2.2 (DUNE Spack stack on CVMFS)             |
| CLHEP          | 2.4.7.1 (from CVMFS)                           |
| Xerces-C       | 3.3.0 (from CVMFS)                             |
| ROOT           | 6.28.12 (from CVMFS, build environment only)   |
| Qt5            | qt5-qtbase-devel (build environment only)      |
| Compiler       | gcc 12.5.0 (from CVMFS)                        |
| Base OS        | Fermilab worker-node EL9 (AlmaLinux 9.8)       |
| Target GPU     | compute capability 7.5 (sm_75)                 |

RiceOpticks is **pinned by commit hash**, not by branch. The `v1.0r_production`
branch has advanced past the commit this image was built from, so building from
the branch tip would produce a different image under the same version label.

The OptiX 9.0.0 headers are present in the image at `/opt/optix/current`. They
are needed at *configure* time by anything that links `Opticks::CSGOptiX`,
because the installed `CSGOptiX.h` includes `<optix.h>` and
`FindOpticksOptiX.cmake` resolves it from `$OPTICKS_OPTIX_PREFIX/include`. The
OptiX *runtime* is not in the image — it ships in your host's NVIDIA driver.

---

## System requirements

This container is **not** self-contained. The host must provide three things,
or the image will not run:

1. **CVMFS**, with these repositories mounted:
   - `/cvmfs/dune.opensciencegrid.org`
   - `/cvmfs/larsoft.opensciencegrid.org`

   Geant4, CLHEP, Xerces-C, ROOT, and the matching gcc runtime all load from
   CVMFS at run time. A host without CVMFS cannot run this image.

   The environment is entered via
   `/cvmfs/dune.opensciencegrid.org/spack/v1.2.2/share/spack/setup-env.sh`.
   Individual packages may resolve into a chained upstream Spack instance; run
   `spack find -p geant4` inside the container to see the actual prefixes in use.

2. **NVIDIA driver R580 or newer**, and an **NVIDIA GPU of Turing architecture
   (compute capability 7.5) or newer** — an RTX 20-series or later, or a
   Turing/Ampere/Ada/Hopper datacenter card. CUDA 13 dropped support for older
   GPUs.

3. **Apptainer** (or Singularity) installed.

Check all three before running anything:

```bash
nvidia-smi                                               # driver >= 580, a Turing+ GPU
ls /cvmfs/dune.opensciencegrid.org /cvmfs/larsoft.opensciencegrid.org
apptainer --version
```

If `nvidia-smi` reports a driver below 580, or `/cvmfs` is missing, the image
will not work on that host. That is a host provisioning issue, not a container
problem.

---

## Getting the image

The built image is published to the GitHub Container Registry as an ORAS
artifact.

```bash
# Large image — point the cache somewhere with room. HPC home dirs usually
# do not have it.
export APPTAINER_CACHEDIR=/scratch/$USER/.apptainer
export APPTAINER_TMPDIR=/scratch/$USER/.apptainer/tmp

apptainer pull riceopticks-v1.0r.sif oras://ghcr.io/nuricelab/riceopticks:v1.0r
```

Use `oras://`, not `docker://` — the artifact is a SIF, not an OCI image.

## Running

```bash
apptainer run --nv --bind /cvmfs riceopticks-v1.0r.sif opticks-info
apptainer run --nv --bind /cvmfs riceopticks-v1.0r.sif bash     # interactive
```

`--nv` exposes the host driver; `--bind /cvmfs` makes the Spack stack visible.
Both are required.

**`apptainer shell` and `apptainer exec` do not run `%runscript`**, so they do
not set up the environment. If you use them, source the setup script yourself:

```bash
apptainer shell --nv --bind /cvmfs riceopticks-v1.0r.sif
Apptainer> source /opt/setup.sh

apptainer exec --nv --bind /cvmfs riceopticks-v1.0r.sif \
    bash -lc 'source /opt/setup.sh; opticks-info'
```

Without that step you get a shell with no Spack, no Geant4, and the wrong
`libstdc++`, and errors that are hard to attribute.

---

## Building the image yourself

Two definition files, built in order:

| File                  | Produces           | Contents                                     |
|-----------------------|--------------------|----------------------------------------------|
| `def/optix-base.def`  | `optix-base.sif`   | fnal-wn-el9 + CUDA 13.0 + OptiX 9.0.0 headers |
| `def/riceopticks.def` | `riceopticks.sif`  | the above + RiceOpticks compiled and installed |

Build host needs Apptainer 1.3 or newer (for `%arguments`), `/cvmfs` mounted,
roughly 40 GB of scratch, and network access to `developer.download.nvidia.com`
and `github.com`. **No GPU is needed to build**, and no manual downloads are
required — the OptiX headers come from NVIDIA's public header-only repository.

```bash
git clone https://github.com/nuRiceLab/riceopticks-container.git
cd riceopticks-container
apptainer build --fakeroot optix-base.sif def/optix-base.def
apptainer build --fakeroot --bind /cvmfs riceopticks-v1.0r.sif def/riceopticks.def
```

`riceopticks.def` bootstraps `From: optix-base.sif`, which resolves relative to
the working directory. Build from the repository root.

Both files expose build arguments, so you can retarget without editing them:

```bash
# build for an Ada GPU (sm_89) at a newer RiceOpticks commit
apptainer build --fakeroot --bind /cvmfs \
    --build-arg COMPUTE_CAPABILITY=89 \
    --build-arg RICEOPTICKS_COMMIT=<40-char-hash> \
    --build-arg RICEOPTICKS_VERSION=v1.1 \
    riceopticks-v1.1.sif def/riceopticks.def
```

`OPTICKS_COMPUTE_CAPABILITY` is baked in at build time, so an image built for
sm_75 is not automatically optimal on newer cards.

To publish a build:

```bash
# one time: a GitHub PAT with write:packages
echo "$GHCR_TOKEN" | apptainer remote login -u <github-user> --password-stdin oras://ghcr.io

apptainer push riceopticks-v1.0r.sif oras://ghcr.io/nuricelab/riceopticks:v1.0r
sha256sum riceopticks-v1.0r.sif > riceopticks-v1.0r.sif.sha256
```

Set the package to public in the organisation's package settings afterwards, or
every user will need their own token.

---

## Citation
@misc{parmaksiz2026acceleratingopticalphotonsimulation,
      title={Accelerating Optical Photon Simulation in DUNE with Opticks}, 
      author={Ilker Parmaksiz and Aaron Higuera and Laura Paulucci and Viktor Pec and Estanislao Forino},
      year={2026},
      eprint={2608.27306},
      archivePrefix={arXiv},
      primaryClass={hep-ex},
      url={https://arxiv.org/abs/2608.27306}, 
}


---

## Licensing

RiceOpticks is a fork of [Opticks](https://bitbucket.org/simoncblyth/opticks/)
by Simon Blyth; see the upstream repository for its terms.

The image contains NVIDIA OptiX 9.0.0 header files obtained from
[NVIDIA/optix-dev](https://github.com/NVIDIA/optix-dev) at tag `v9.0.0`. Those
headers are individually licensed under either BSD-3-Clause or an NVIDIA
proprietary licence, governed by the NVIDIA DesignWorks agreement in that
repository's `LICENSE.txt`. No OptiX runtime library is redistributed; the
implementation is supplied by the host NVIDIA driver.

CUDA is installed from NVIDIA's package repository under the CUDA Toolkit
end-user licence agreement.
