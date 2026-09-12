# Fedora-44-NVIDIA-RTX-3050-Secure-Boot
This guide explains how to install the NVIDIA driver on Fedora 44 while keeping UEFI Secure Boot enabled.  Tested on:  Fedora 44 ASUS TUF Gaming F15 FX507ZC4 NVIDIA GeForce RTX 3050 Mobile NVIDIA driver 610.57.04 Secure Boot enabled GNOME / KDE should both work Dual boot with Windows 11
Absolutely. Here is a **clean GitHub-ready README** based on the setup we actually used, including the Secure Boot part that caused the problem.

# Fedora 44 + NVIDIA RTX 3050 + Secure Boot

This guide explains how to install the NVIDIA driver on **Fedora 44** while keeping **UEFI Secure Boot enabled**.

Tested on:

* Fedora 44
* ASUS TUF Gaming F15 FX507ZC4
* NVIDIA GeForce RTX 3050 Mobile
* NVIDIA driver `610.57.04`
* Secure Boot **enabled**
* GNOME / KDE should both work
* Dual boot with Windows 11

---

## 1. Update Fedora

After installing Fedora, update the system:

```bash
sudo dnf upgrade --refresh
```

Reboot:

```bash
sudo systemctl reboot -i
```

---

## 2. Check Secure Boot

Check that Secure Boot is enabled:

```bash
mokutil --sb-state
```

Expected:

```text
SecureBoot enabled
```

**Do not disable Secure Boot** if you want to keep it enabled for your Windows/Fedora dual boot.

---

## 3. Add RPM Fusion

Install the RPM Fusion Free and Nonfree repositories:

```bash
sudo dnf install \
https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```

No reboot is required here.

---

## 4. Install the NVIDIA driver

Install `akmod-nvidia`:

```bash
sudo dnf install akmod-nvidia
```

Check that the NVIDIA kernel module exists:

```bash
modinfo -F version nvidia
```

For this setup it returned:

```text
610.57.04
```

At this point the driver can exist but **still not work with Secure Boot**, because the kernel module needs to be signed.

---

# Secure Boot Setup

## 5. Create an akmods signing key

Run:

```bash
sudo kmodgenca -a
```

On a fresh installation this creates a key pair.

You should see something similar to:

```text
SUCCESS!
Public Key (Certificate) created at:
/etc/pki/akmods/certs/fedora_XXXXXXXXXX_XXXXXXXX.der

Private Key created at:
/etc/pki/akmods/private/fedora_XXXXXXXXXX_XXXXXXXX.priv
```

### Important

If you see:

```text
WARNING: EXISTING KEY PAIR.
Please specify argument '--force' to overwrite the existing key pair.
```

**Do NOT use `--force` just because you see this.**

An existing key may already be present and should normally be reused.

---

## 6. Enroll the key into Secure Boot

Import the public key:

```bash
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
```

You will be asked to create a temporary password.

Remember this password because you will need it during the next reboot.

**Do not reboot yet.**

---

## 7. Rebuild the NVIDIA kernel module

Now rebuild the NVIDIA module so that it gets signed using the akmods key:

```bash
sudo akmods --rebuild --force
```

Check whether the module has been signed:

```bash
modinfo -F signer nvidia
```

You should get something similar to:

```text
fedora_1789229676_e1288007
```

The exact name will be different on every installation.

If this command returns a key name, the NVIDIA module is signed.

---

# 8. Reboot and enroll the MOK

Reboot:

```bash
sudo systemctl reboot -i
```

During boot, you should see:

**Perform MOK management**

Choose:

```text
Enroll MOK
```

Then:

```text
Continue
```

Then:

```text
Yes
```

Enter the temporary password you created with:

```bash
sudo mokutil --import ...
```

Then reboot.

This step is important because it tells the UEFI Secure Boot system to trust the key used to sign the NVIDIA kernel module.

---

# 9. Check that NVIDIA loaded

After Fedora starts, run:

```bash
lsmod | grep nvidia
```

You should see modules such as:

```text
nvidia
nvidia_drm
nvidia_modeset
nvidia_uvm
```

You can also check the signature again:

```bash
modinfo -F signer nvidia
```

---

# 10. Install NVIDIA utilities

`nvidia-smi` may not be installed by `akmod-nvidia` alone.

Install it with:

```bash
sudo dnf install xorg-x11-drv-nvidia-cuda
```

Then run:

```bash
nvidia-smi
```

A working installation should show your NVIDIA GPU, driver version and GPU memory.

For the RTX 3050 used while testing this guide, the result showed:

```text
NVIDIA-SMI 610.57.04
KMD Version: 610.57.04
CUDA UMD Version: 13.3
```

and:

```text
NVIDIA GeForce RTX 3050
```

---

# Troubleshooting

## `modinfo -F signer nvidia` is empty

This means the module exists but does not have a visible signing certificate.

Try:

```bash
sudo akmods --rebuild --force
```

Then:

```bash
modinfo -F signer nvidia
```

If it is still empty, check that the MOK key has been enrolled:

```bash
sudo mokutil --test-key /etc/pki/akmods/certs/public_key.der
```

If it says:

```text
is already enrolled
```

the key is enrolled.

---

## `modprobe nvidia` says `Key was rejected by service`

Example:

```text
modprobe: ERROR: could not insert 'nvidia': Key was rejected by service
```

This normally indicates a **Secure Boot signing/trust problem**.

Check:

```bash
mokutil --sb-state
```

and:

```bash
modinfo -F signer nvidia
```

Make sure the signing key has been enrolled through the MOK screen during boot.

---

## `nvidia-smi` says it cannot communicate with the driver

First check:

```bash
lsmod | grep nvidia
```

If nothing appears, try:

```bash
sudo modprobe nvidia
```

Then check:

```bash
nvidia-smi
```

Also check which driver is controlling the GPU:

```bash
lspci -k -s 01:00.0
```

A working NVIDIA setup should eventually show:

```text
Kernel driver in use: nvidia
```

and not:

```text
Kernel driver in use: nouveau
```

---

## `nouveau` is being used

Check:

```bash
lspci -k -s 01:00.0
```

If you see:

```text
Kernel driver in use: nouveau
```

NVIDIA cannot normally take control of the GPU while nouveau has already claimed it.

**Do not run:**

```bash
sudo modprobe -r nouveau
```

while using the graphical desktop if it says:

```text
Module nouveau is in use
```

Instead, fix the boot configuration carefully and reboot.

---

# Important lessons from this installation

### Don't disable Secure Boot just to make NVIDIA work

It is possible to run NVIDIA with Secure Boot enabled. The NVIDIA kernel module needs to be signed with a key trusted by Secure Boot.

### Don't use `kmodgenca --force` unnecessarily

If `kmodgenca -a` says an existing key pair already exists, don't overwrite it unless you specifically know why you need a new key.

### Check the signature before rebooting

This is the most useful check:

```bash
modinfo -F signer nvidia
```

If it returns the akmods key name, the module is signed.

### Don't blacklist nouveau too early

The previous installation showed why this matters. If NVIDIA isn't ready and nouveau is blacklisted, you can end up with a system that doesn't boot into the graphical desktop properly.

Get this working first:

```bash
modinfo -F signer nvidia
```

then after reboot:

```bash
lsmod | grep nvidia
```

and finally:

```bash
nvidia-smi
```

---

# Quick version

For someone who already understands Fedora, the process is basically:

```bash
sudo dnf upgrade --refresh

sudo dnf install \
https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

sudo dnf install akmod-nvidia

sudo kmodgenca -a

sudo mokutil --import /etc/pki/akmods/certs/public_key.der

sudo akmods --rebuild --force

modinfo -F signer nvidia
```

If the last command shows the signing key, reboot, **Enroll MOK**, enter the password, reboot again, then:

```bash
sudo dnf install xorg-x11-drv-nvidia-cuda

nvidia-smi
```

If `nvidia-smi` shows the GPU, the installation is complete. ✅

**One important note for future Fedora releases:** package names and NVIDIA driver versions can change, so the exact version `610.57.04` in this guide is an example from this installation, not something that should be manually forced on future releases.
