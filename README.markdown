# nvtop-spark

A maintained fork of [Syllo/nvtop](https://github.com/Syllo/nvtop), focused on
NVIDIA DGX Spark / GB10 memory reporting and a compact memory-bar display.
This is a community fork, **not NVIDIA-official software**. NVTOP's upstream
history, contributors and licensing are retained; the original documentation
continues below.

## Spark changes and provenance

This fork starts at upstream commit
[`71094a9f7dfbfe7cbe9faebfdf055e580a034283`](https://github.com/Syllo/nvtop/commit/71094a9f7dfbfe7cbe9faebfdf055e580a034283),
a later baseline than the **3.3.2 release**. The version command still reports
`3.3.2`; use `git rev-parse HEAD` to identify this fork's actual revision.

The shared-memory implementation is upstream work, not newly invented here:

- [Santosh Bhavani's unified-memory contribution, PR #411](https://github.com/Syllo/nvtop/pull/411), included in the 3.3.0 release.
- [Billions of Yuan (`redyuan43`), PR #466](https://github.com/Syllo/nvtop/pull/466), including the NVIDIA UMA fallback and correction to GPU-process memory summation.
- [Maxime Schmitt's subsequent generic process-accounting integration](https://github.com/Syllo/nvtop/commit/5d164b8dd631931486b38754aaf5bc29da1bc6ff), included in our pinned baseline.

On top of that baseline, this fork carries two small local changes, developed
and validated on DGX Spark:

1. **NVML memory-info v2 compatibility:** encode both structure-version fields
   as `sizeof(struct) | (2U << 24)` rather than plain `2`. On driver 580.178.04,
   the original call failed before the shared-memory fallback could be selected.
   The corrected call allows upstream's UMA detection/accounting to work.
2. **Compact memory labels:** show `52.8/121.7Gi` rather than
   `52.762Gi/121.688Gi`, with one decimal place and one trailing unit. The
   unknown-used label is similarly compact. This is presentation only; the
   underlying byte counts and bar percentages are unchanged.

These local changes were not imported from a third-party nvtop fork or an NVML
preload shim. The source is Syllo/nvtop plus the changes above. See
[COPYING](COPYING) for the retained upstream license.

## What the GB10 memory values mean

GB10 shares RAM between the CPU and GPU; it has no separate VRAM pool.

| Value | Source |
|---|---|
| Total | Linux `/proc/meminfo` `MemTotal` (about 121.7 GiB on the tested Spark). |
| Used | Sum of valid NVML-reported GPU-process allocations, using upstream nvtop's process accounting. |
| Free (snapshot output) | Linux `MemAvailable`, not simply total minus GPU-used. |

CPU allocations are not part of the GPU-used sum, so used plus free need not
sum to total. These are reported allocations, not an exact physical-memory
ownership measurement. Unreported allocations cannot be counted. The patch does
not make `nvidia-smi`'s device-wide framebuffer query supported.

## Build and install on DGX Spark

Tested on NVIDIA DGX Spark OS 7.6.0 / Ubuntu 24.04 ARM64 with GCC 13.3,
CMake 3.28 and NVIDIA driver 580.178.04. Use the existing NVIDIA driver; no driver
replacement, CUDA toolkit, preload shim or root privileges are needed to run it.
The preset requires CMake 3.21 or newer and selects NVIDIA-only Release settings
through upstream's existing CMake options.

```sh
sudo apt-get install build-essential cmake git libncurses-dev libudev-dev
git clone https://github.com/kingsleyfaulkner/nvtop-spark.git
cd nvtop-spark
cmake --preset spark
cmake --build --preset spark
./build/spark/src/nvtop
```

For a system-wide installation, after testing the build:

```sh
sudo cmake --install build/spark
hash -r
nvtop
```

This uses `/usr/local` and installs `nvtop`, its manpage and desktop metadata.
It does not remove the distribution's package, but `/usr/local/bin` normally
precedes `/usr/bin` on PATH. It **replaces an existing local installation** at
the same prefix; back up any private build you want to keep first.
To install for just your account instead, use
`cmake --install build/spark --prefix "$HOME/.local"` and ensure
`$HOME/.local/bin` is on PATH. Do not grant the binary setuid or extra capabilities.

The preset does not run a full test suite. Focused validation includes a live
GB10 JSON snapshot (`./build/spark/src/nvtop -s`), checking GPU-used against the
process-allocation sum, and checking the compact label in an interactive terminal.

### Update or uninstall

From an unchanged checkout, update this fork's `spark` branch and rebuild:

```sh
git pull --ff-only
cmake --preset spark
cmake --build --preset spark
sudo cmake --install build/spark
```

Review `build/spark/install_manifest.txt` to see the exact installed files.
To remove a system-wide installation made by this build, after reviewing that
manifest:

```sh
sudo xargs -d '\n' rm -- < build/spark/install_manifest.txt
hash -r
```

For a user-prefix installation, omit `sudo`. Keep the manifest from the relevant
installation; it records the chosen prefix. Distribution packages are unaffected.

---

# Upstream NVTOP documentation

What is NVTOP?
--------------

NVTOP stands for Neat Videocard TOP, a (h)top like task monitor for GPUs and
accelerators. It can handle multiple GPUs and print information about them in a
htop-familiar way.

Currently supported vendors are AMD (Linux amdgpu driver), Apple (limited M1 &
M2 support), Huawei (Ascend), Intel (Linux i915/Xe drivers), NVIDIA (Linux
proprietary divers), Qualcomm Adreno (Linux MSM driver), Broadcom VideoCore (Linux v3d driver),
Rockchip, MetaX (MXSML driver), Enflame (Linux EFML driver), Tenstorrent (Linux tt-kmd driver).
Rockchip, MetaX (MXSML driver), Enflame (Linux EFML driver), Iluvatar CoreX (ixML / libixml).

Because a picture is worth a thousand words:

![NVTOP interface](/screenshot/NVTOP_ex1.png)

Table of Contents
-----------------

- [NVTOP Options and Interactive Commands](#nvtop-options-and-interactive-commands)
  - [Interactive Setup Window](#interactive-setup-window)
  - [Saving Preferences](#saving-preferences)
  - [NVTOP Manual and Command line Options](#nvtop-manual-and-command-line-options)
- [GPU Support](#gpu-support)
  - [AMD](#amd)
  - [Intel](#intel)
  - [NVIDIA](#nvidia)
  - [Adreno](#adreno)
  - [Apple](#apple)
  - [Ascend](#ascend) (only tested on 910B)
  - [Iluvatar CoreX](#iluvatar-corex)
  - [VideoCore](#videocore)
  - [Rockchip](#rockchip)
  - [MetaX](#metax)
  - [Enflame](#enflame)
  - [Tenstorrent](#tenstorrent)
- [Build](#build)
- [Distribution Specific Installation Process](#distribution-specific-installation-process)
  - [Ubuntu / Debian](#ubuntu--debian)
    - [Ubuntu Impish (21.10) / Debian buster (stable) and more recent (stable)](#ubuntu-impish-2110-debian-buster-stable-and-more-recent)
  - [Fedora / Red Hat / CentOS](#fedora--red-hat--centos)
  - [OpenSUSE](#opensuse)
  - [Arch Linux](#arch-linux)
  - [Gentoo](#gentoo)
  - [AppImage](#appimage)
  - [Snap](#snap)
  - [Conda-forge](#conda-forge)
  - [Docker](#docker)
  - [WSL2](#wsl2)
- [NVTOP Build](#nvtop-build)
- [Troubleshoot](#troubleshoot)
- [License](#license)

NVTOP Options and Interactive Commands
--------------------------------------
### Interactive Setup Window

NVTOP has a builtin setup utility that provides a way to specialize the interface to your needs.
Simply press ``F2`` and select the options that are the best for you.

In the ``Chart`` section you can choose which metrics are plotted, including GPU and memory
utilization, temperature, power, clocks, and the **PCIe RX / TX load** (the receive and transmit
throughput as a percentage of the maximum link bandwidth).

![NVTOP Setup Window](/screenshot/Nvtop-config.png)

### Saving Preferences

You can save the preferences set in the setup window by pressing ``F12``.
The preferences will be loaded the next time you run ``nvtop``.

### NVTOP Manual and Command line Options

NVTOP comes with a manpage!
```bash
man nvtop
```
For quick command line arguments help
```bash
nvtop -h
nvtop --help
```

GPU Support
-----------

### AMD

NVTOP supports AMD GPUs using the `amdgpu` driver and the legacy `radeon` driver (legacy GPUs, limited support) through the exposed DRM and
sysfs interface.

The radeon provides limited metrics compared to amdgpu.

AMD introduced the fdinfo interface in kernel 5.14 ([browse kernel
source](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/amd/amdgpu/amdgpu_fdinfo.c?h=linux-5.14.y)).
Hence, you will need a kernel with a version greater or equal to 5.14 to see the
processes using AMD GPUs.

Support for recent GPUs are regularly mainlined into the linux kernel, so please
use a recent-enough kernel for your GPU.

### Intel

NVTOP supports Intel GPUs using the `i915` or `xe` linux driver.

Intel introduced the fdinfo interface in kernel 5.19 ([browse kernel
source](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/i915/i915_drm_client.c?h=linux-5.19.y)).
Hence, you will need a kernel with a version greater or equal to 5.19 to see the
processes using Intel GPUs.

Intel requires CAP_PERFMON or CAP_SYS_ADMIN capabilities to access the total memory usage, and an accurate GPU frequency.
you can run `sudo setcap cap_perfmon=ep $(which nvtop)` to grant the necessary permissions or run nvtop as root.

### NVIDIA

The *NVML library* does not support some of the queries for GPUs coming before the
Kepler microarchitecture. Anything starting at GeForce 600, GeForce 800M and
successor should work fine. For more information about supported GPUs please
take a look at the [NVML documentation](http://docs.nvidia.com/deploy/nvml-api/nvml-api-reference.html#nvml-api-reference).

The extra GPU info bar (`nvtop -i`) shows the dynamic parameters NVML does not
report, such as the XBAR, SYS and NVD clocks, and `nvtop -X` also shows the
static specs (shader cores, L2 cache, execution engines) and the secondary HUB,
HOST, DISP, MSD and UTILS clocks. The clock domains are read through the *NvAPI
library* (`libnvidia-api.so.1`) that recent drivers install alongside NVML;
without it the rest of the interface is unaffected and the bar takes no room on
a GPU that reports nothing.

### Adreno

NVTOP supports Adreno GPUs using the `msm` linux driver.

msm introduced the fdinfo interface in kernel 6.0 ([browse kernel
source](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/msm/msm_drv.c?h=linux-6.0.y)).
Hence, you will need a kernel with a version greater or equal to 6.0 to see the
processes using Adreno GPUs.

### Apple

NVTOP includes some initial support for Apple using Metal. This is only supported when building for Apple, and when building for Apple only this vendor is supported.

**APPLE SUPPORT STATUS**
- Apple support is still being worked on. Some bugs and limitations may apply.

### Ascend

NVTOP supports Ascend through the DCMI API (tested on 910B3 hardware). It
uses the logical-device API when available and falls back to the legacy
card/device API. Read-only telemetry is reported when supported by the
installed driver; unavailable values remain unset. Enable it with
`-DASCEND_SUPPORT=ON`, and set `-DASCEND_ROOT=/path/to/Ascend` when CANN is
installed outside the standard locations.

### Iluvatar CoreX

NVTOP supports Iluvatar CoreX GPUs through the ixML library.

The backend dynamically loads `libixml.so` from `/usr/local/corex/lib`,
`/usr/local/corex/lib64`, or the default dynamic loader search path. The ixML
runtime exposes an NVML-compatible API surface used by NVTOP to query device,
power, PCIe, clock, temperature, memory, and process information.

### VideoCore

NVTOP supports VideoCore (testing on raspberrypi 4B).

Supports GPU frequency, temperature, utilization, per-process utilization, GPU memory usage, and H264 decoding utilization.

On non-raspberry pi os, you need to use the `linux-rpi 6.12.y` kernel and above, and ensure the presence of the `/dev/vcio` device.

### Rockchip

NVTOP supports Rockchip (testing on orangepi 5 plus).

Supports NPU frequency, temperature, utilization.

### MetaX

NVTOP supports MetaX (testing on MXC500) by MXSML LIBRARY.

For more information about GPUs please take a look at the [METAX documentation](https://developer.metax-tech.com/doc/index)

### Enflame

NVTOP supports Enflame GCUs (testing on Enflame S60, Enflame L300 and Enflame L600) by EFML LIBRARY

GCU, which refers to General Compute Unit, is a type of accelerator card that is used to perform general-purpose computing tasks just like GPGPU.

### Tenstorrent

NVTOP supports Tenstorrent AI accelerators (Blackhole, Wormhole, Grayskull) through the [tt-kmd](https://github.com/tenstorrent/tt-kmd) kernel driver.

Supports temperature, power draw, AI clock, fan RPM, PCIe link info, and process listing. No external libraries required -- all data is read from sysfs, hwmon, and procfs.

Build
-----

Several libraries are required in order for NVTOP to display GPU info:

* The *ncurses* library driving the user interface.
  * This makes the screen look beautiful.
* For NVIDIA: the *NVIDIA Management Library* (*NVML*) which comes with the GPU driver.
  * This queries the GPU for info.
  * The *NvAPI library* (`libnvidia-api.so.1`), also from the driver, is used
    when present for the clock domains NVML does not report. It is optional.
* For AMD: the libdrm library used to query AMD GPUs through the kernel driver.
* For METAX: the *MetaX System Management Library* (*MXSML*) which comes with the GPU driver.
  * This queries the GPU for info.
* For Enflame: the *Enflame Management Library* (*EFML*) which comes with the GCU driver.
* For Iluvatar CoreX: the *ixML* runtime library (`libixml.so`) which comes with the driver.
  * This backend loads the library dynamically at runtime.

## Distribution Specific Installation Process

### Ubuntu / Debian

If your distribution provides the snap utility, follow the [snap installation process](#snap) to obtain an up-to-date version of `nvtop`.

A standalone application is available as [AppImage](#appimage).

#### Ubuntu Focal (20.04), Debian buster (stable) and more recent

```bash
sudo apt install nvtop
```

#### Ubuntu PPA

A [PPA supporting Ubuntu 20.04 and newer](https://launchpad.net/~quentiumyt/+archive/ubuntu/nvtop) is provided by
[Quentin Lienhardt](https://github.com/QuentiumYT) that offers an up-to-date version of `nvtop`, enabled for NVIDIA, AMD and Intel.

```bash
sudo add-apt-repository ppa:quentiumyt/nvtop
sudo apt install nvtop
```

#### Older

- AMD and Intel Dependencies
  ```bash
  sudo apt install libdrm-dev libsystemd-dev
  # Ubuntu 18.04
  sudo apt install libudev-dev
  ```

- NVIDIA Dependency
  - NVIDIA drivers (see [Ubuntu Wiki](https://help.ubuntu.com/community/BinaryDriverHowto/Nvidia) or [Ubuntu PPA](https://launchpad.net/~graphics-drivers/+archive/ubuntu/ppa) or [Debian Wiki](https://wiki.debian.org/NvidiaGraphicsDrivers#NVIDIA_Proprietary_Driver))

- NVTOP Dependencies
  - CMake, ncurses and Git
  ```bash
  sudo apt install cmake libncurses5-dev libncursesw5-dev git
  ```

- NVTOP
  - Follow the [NVTOP Build](#nvtop-build)


### Fedora / Red Hat / CentOS

A standalone application is available as [AppImage](#appimage).

#### Fedora 36 and newer

- ```bash
  sudo dnf install nvtop
  ```

#### Red Hat Enterprise Linux 8 and 9

- ```bash
  sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(rpm -E %{rhel}).noarch.rpm
  sudo dnf install nvtop
  ```

#### CentOS Stream, Rocky Linux, AlmaLinux

- ```bash
  sudo dnf install -y epel-release
  sudo dnf install nvtop
  ```

#### Build process for Fedora / Red Hat / CentOS:

- AMD and Intel Dependencies
  ```bash
  sudo dnf install libdrm-devel systemd-devel
  ```

- NVIDIA Dependency
  - NVIDIA drivers, **CUDA required for nvml libraries** (see [RPM Fusion](https://rpmfusion.org/Howto/NVIDIA))

- NVTOP Dependencies
  - CMake, ncurses, C++ and Git
  ```bash
  sudo dnf install cmake ncurses-devel git gcc-c++
  ```

- NVTOP
  - Follow the [NVTOP Build](#nvtop-build)

### OpenSUSE

A standalone application is available as an [AppImage](#appimage).

Build process for OpenSUSE:

- AMD Dependency
  ```bash
  sudo zypper install libdrm-devel
  ```

- NVIDIA Dependency
  - NVIDIA drivers (see [SUSE Support Database](https://en.opensuse.org/SDB:NVIDIA_drivers))

- NVTOP Dependencies
  - CMake, ncurses and Git
    ```bash
    sudo zypper install cmake ncurses-devel git
    ```

- NVTOP
  - Follow the [NVTOP Build](#nvtop-build)

### Arch Linux

- ```bash
  sudo pacman -S nvtop
  ```

### Gentoo

- ```bash
  sudo emerge -av nvtop
  ```

### AppImage

An AppImage is a standalone application. Just download the AppImage, make it executable and run it!

- Go to the [release page](https://github.com/Syllo/nvtop/releases/latest) and download `nvtop-x86_64.AppImage`

- ```bash
  # Go to the download location ** The path may differ on your system **
  cd $HOME/Downloads
  # Make the AppImage executable
  chmod u+x nvtop-x86_64.AppImage
  # Enjoy nvtop
  ./nvtop-x86_64.AppImage
  ```

If you are curious how that works, please visit the [AppImage website](https://appimage.org/).

### Snap

- ```bash
  snap install nvtop
  # Add the capability to kill processes inside nvtop
  snap connect nvtop:process-control
  # Add the capability to inspect GPU information (fan, PCIe, power, etc)
  snap connect nvtop:hardware-observe
  # AMDGPU process list support (read /proc/<pid>)
  snap connect nvtop:system-observe
  # Temporary workaround to get per-process GPU usage (read /proc/<pid>/fdinfo)
  snap connect nvtop:kubernetes-support
  ```

Notice: The connect commands allow

### Conda-forge

A [conda-forge feedstock for `nvtop`](https://github.com/conda-forge/nvtop-feedstock) is available.

#### conda / mamba / miniforge

```bash
conda install --channel conda-forge nvtop
```

#### pixi

```bash
pixi global install nvtop
```

### Docker

- NVIDIA drivers (same as above)

- [nvidia-docker](https://github.com/NVIDIA/nvidia-docker) (See [Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker))

- ```bash
  git clone https://github.com/Syllo/nvtop.git && cd nvtop
  sudo docker build --tag nvtop .
  sudo docker run -it --rm --runtime=nvidia --gpus=all --pid=host nvtop
  ```

### WSL2

Nvtop support in WSL2 mostly boils down to GPU/Accelerator vendor support of
WSL2. Nvtop should work out of the box for the following configurations:

- NVIDIA: Please refer to [NVIDIA's user guide to get started with CUDA on
WSL2](https://docs.nvidia.com/cuda/wsl-user-guide/index.html#getting-started-with-cuda-on-wsl)
to avoid overiding the driver exposed by WSL2 (from Windows) with the NVIDIA
Linux drivers.

## NVTOP Build

```bash
git clone https://github.com/Syllo/nvtop.git
mkdir -p nvtop/build && cd nvtop/build
cmake .. -DNVIDIA_SUPPORT=ON -DAMDGPU_SUPPORT=ON -DINTEL_SUPPORT=ON
make

# Install globally on the system
sudo make install

# Alternatively, install without privileges at a location of your choosing
# cmake .. -DNVIDIA_SUPPORT=ON -DAMDGPU_SUPPORT=ON -DINTEL_SUPPORT=ON -DCMAKE_INSTALL_PREFIX=/path/to/your/dir
# make
# make install
```

If you use **conda** as environment manager and encounter an error while building NVTOP, try `conda deactivate` before invoking `cmake`.

The build system supports multiple build types (e.g. -DCMAKE_BUILD_TYPE=RelWithDebInfo):

* Release: Binary without debug info
* RelWithDebInfo: Binary with debug info
* Debug: Compile with warning flags and address/undefined sanitizers enabled (for development purposes)

Troubleshoot
------------

- The plot looks bad:
  - Verify that you installed the wide character version of the ncurses library (libncurses**w**5-dev for Debian / Ubuntu), clean the build directory and restart the build process.
- If `nvtop` exits with `ncurses: cannot initialize terminal type ($TERM="unknown")`, ensure that `$TERM` is set to a valid terminal type such as `xterm-256color`.
- **Putty**: Tell putty not to lie about its capabilities (`$TERM`) by setting the field ``Terminal-type string`` to ``putty`` in the menu
  ``Connection > Data > Terminal Details``.
- `NO GPU to monitor.` for NVIDIA GPUs:
  - `nvtop` loads a shared library named `libnvml.so` (shipped with the NVIDIA
  drivers) to querry device information. If the library is not present, nvtop
  will not be able to monitor your NVIDIA device.
  - On `WSL2`, the installation instructions are slightly different since the
  driver is being exposed by Windows to the virtual machine (see instruction
  link in [the WSL2 section](#wsl2)). If you install the NVIDIA linux drivers
  inside WSL2, you may encounter a version mismatch error or nvtop may silently
  not work at all.

License
-------

NVTOP is licensed under the GPLv3 license or any later version.
You will find a copy of the license inside the COPYING file of the repository or
at the GNU website <[www.gnu.org/licenses/](http://www.gnu.org/licenses/)>.
