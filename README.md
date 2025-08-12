## Problem Overview

After upgrading a Tuxedo laptop from **Ubuntu 22.04 LTS (Jammy)** to **Ubuntu 24.04 LTS (Noble)**, the system experienced NVIDIA driver conflicts:

- The kernel was loading the **570.x** driver from Tuxedo's repository,
- `nvidia-smi` was reporting **580.x** from NVIDIA/CUDA repository,
- Installing new packages failed due to missing dependencies or `held broken packages`,
- Multiple repositories were active, including `ubuntu2204`, `ubuntu2404`, and `jammy`.

**Result:**
- The wrong NVIDIA driver was loaded after boot,
- `prime-select` was inconsistent,
- Installing driver 580 was impossible without cleaning up the configuration.

---

## Goals of the Fix

1. Remove leftover repositories from Ubuntu 22.04 / Jammy.
2. Unlock and clean up NVIDIA packages that were on hold.
3. Add the correct NVIDIA/CUDA repository for Ubuntu 24.04.
4. Force install **580.65.06** NVIDIA driver.
5. Enable NVIDIA GPU in PRIME mode.
6. Prevent automatic downgrade to Tuxedo’s 570 driver.

---

## Step-by-Step Guide

### 1. Check OS version and active APT sources

```bash
lsb_release -a
grep -RniE 'jammy|noble|ubuntu2204|ubuntu2404' /etc/apt/sources.list /etc/apt/sources.list.d/*.list
If you see jammy or ubuntu2204 entries — they must be removed or disabled.

2. Remove old CUDA 2204 repository

Copy
sudo rm -f /etc/apt/sources.list.d/cuda-ubuntu2204-x86_64.list
3. Unhold all NVIDIA packages

sudo apt-mark unhold 'libnvidia-*' 'nvidia-*' 'xserver-xorg-video-nvidia*'
4. Add the correct CUDA/NVIDIA repository for Ubuntu 24.04

sudo mkdir -p /usr/share/keyrings
curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub \
  | sudo gpg --dearmor -o /usr/share/keyrings/cuda-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/cuda-archive-keyring.gpg] \
https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /" \
  | sudo tee /etc/apt/sources.list.d/cuda-ubuntu2404-x86_64.list
Update the package list:



sudo apt update
5. Install NVIDIA driver 580


sudo apt install -y \
  nvidia-driver-580-open \
  nvidia-dkms-580 \
  nvidia-kernel-common-580 \
  nvidia-utils-580 \
  libnvidia-compute-580:amd64
6. Verify the installed version

modinfo nvidia | grep ^version
nvidia-smi
readlink -f /usr/lib/x86_64-linux-gnu/libnvidia-ml.so.1
Expected output: 580.65.06.

7. Enable NVIDIA GPU in PRIME mode

sudo prime-select nvidia
prime-select query
Reboot the system:


sudo reboot
After reboot:




nvidia-smi
GPU should be active (Disp.A On).

8. Block NVIDIA packages from Tuxedo repo
Create an APT preferences file:


echo 'Package: *nvidia*
Pin: origin "deb.tuxedocomputers.com"
Pin-Priority: -1' | sudo tee /etc/apt/preferences.d/block-tuxedo-nvidia.pref
Update:

bash
Always show details

Copy
sudo apt update
From now on, APT will never install NVIDIA packages from the Tuxedo repository.

Final Result
The system is running NVIDIA driver 580.65.06,

PRIME mode is set to nvidia,

No version conflicts,

Tuxedo repository still available for other packages, but NVIDIA packages are blocked.

Notes
If a kernel update causes issues with the NVIDIA module, run:


sudo dkms autoinstall
On hybrid laptops, you can switch back to Intel GPU with:

sudo prime-select intel   #or prime-select nvidia
sudo reboot
