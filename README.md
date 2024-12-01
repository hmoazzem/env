# Passing-through PCI devices (GPU, NVMe SSD, NIC, etc) to Virtual Machine (VM)

PCI passthrough enables a virtual machine (VM) of any OS to have its dedicated hardware (eg a GPU) with near-native performance. See [PCI_passthrough_via_OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) for more.


> We want to passthrough a GPU and an NVMe SSD to a Windows 11 VM (for gaming).

It's possible to passthrough an existing bare-metal
Windows installation storage (eg NVMe SSD), enabling flexibility to run the same Windows both as a VM and directly on hardware. That's what we're doing.

So far I've got:

- [AMD Adrenalin](https://www.amd.com/en/products/software/adrenalin.html) to detect GPU on Windows 11 VM
- PyTorch to detect GPU (CUDA/ROCm) on RHEL 9.4 VM

|                                                    |                                                  |
|----------------------------------------------------|--------------------------------------------------|
| ![adrenalin](./screenshots/win11-pt-adrenalin.png) | ![dxdiag](./screenshots/win11-pt-dxdiag.png)     |


**But the GPU display output is still having rendering issue**

|                                                     |                                                               |
|-----------------------------------------------------|---------------------------------------------------------------|
| ![boot-uefi](./screenshots/win11-pt-boot-uefi.jpeg) | ![display-glitch](./screenshots/win11-pt-display-glitch.jpeg) |

The left screen shows the host display (via motherboard HDMI) running Windows in the virt-manager GUI. The right shows the passthrough GPU's output to the VM, which appears to have rendering issues, likely related to framebuffer settings.

## Host (Linux)

I'm running Fedora 41 Workstatation on AMD Ryzen 7950X, AMD Radeon RX 7900 XTX and AsRock X870E Taichi Lite.

1. Enable Virtualization and IOMMU on BIOS/UEFI. Next ensure Virtualization and IOMMU from host shell.

```sh
sudo dmesg | grep -i -e DMAR -e IOMMU
```

Also check IOMMU devices by group

```sh
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

For passthrough, all devices within an IOMMU group must be bound to a VFIO device driver eg `vfio-pci` for PCI devices.

2. Find the [BDF](https://wiki.xenproject.org/wiki/Bus:Device.Function_(BDF)_Notation) id, `vendor_id:device_id` of the GPU to be passed-through

```sh
lspci -nnv | grep -A10 -E 'VGA|Audio|NVMe'
```

Output looks somewhat like

```sh
03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 [Radeon RX 7900 XT/7900 XTX/7900 GRE/7900M] [1002:744c] (rev c8) (prog-if 00 [VGA controller])
        Subsystem: Sapphire Technology Limited NITRO+ RX 7900 XTX Vapor-X [1da2:e471]
        Flags: bus master, fast devsel, latency 0, IRQ 186, IOMMU group 15
        Kernel driver in use: amdgpu
        Kernel modules: amdgpu

03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
        Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
        Flags: bus master, fast devsel, latency 0, IRQ 176, IOMMU group 16
        Kernel driver in use: snd_hda_intel
        Kernel modules: snd_hda_intel

6e:00.0 Non-Volatile memory controller [0108]: Micron/Crucial Technology P3 Plus NVMe PCIe SSD (DRAM-less) [c0a9:5421] (rev 01) (prog-if 02 [NVM Express])
        Subsystem: Micron/Crucial Technology Device [c0a9:5021]
        Flags: bus master, fast devsel, latency 0, IRQ 24, IOMMU group 19
        Kernel driver in use: nvme
        Kernel modules: nvme
```

GPUs typically have a VGA and an Audio *Function* / component; both need to be bound to `vfio-pci` kernel driver.

> **On host we must ensure `Kernel driver in use: vfio-pci` (as opposed to `amdgpu`, `nvidia`, `snd_hda_intel`, `nvme` etc) for the devices**.

2. Update `GRUB_CMDLINE_LINUX` in `/etc/default/grub`
```sh
# truncated
GRUB_CMDLINE_LINUX="APPEND TO EXISTING PARAMS amd_iommu=on amd_iommu=pt vfio-pci.ids=1002:744c,1002:ab30,c0a9:5421"
# truncated
```

For Intel CPU, set `intel_iommu=on`. I've also seen in several guides `video=efifb:off`
which doesn't seem to change anything, but I suspect it's relevant for framebuffers issues.

3. Configure VFIO for device passthrough

```sh
cat <<EOF | sudo tee /etc/modprobe.d/vfio.conf 
options vfio-pci ids=1002:744c,1002:ab30,c0a9:5421
options vfio_iommu_type1 allow_unsafe_interrupts=1
softdep drm pre: vfio-pci
EOF
```

If needed, blacklist default/conflicting drivers

```sh
cat <<EOF | sudo tee /etc/modprobe.d/vfio-blacklist.conf 
blacklist amdgpu
blacklist snd_hda_intel
EOF
```

4. Configure `dracut` to include VFIO drivers in initramfs
```sh
cat <<EOF | sudo tee /etc/dracut.conf.d/00-vfio.conf 
force_drivers+=" vfio_pci vfio vfio_iommu_type1 "
EOF
```

5. Regenerate initramfs with VFIO drivers
```sh
sudo dracut -f --kver $(uname -r)
```

6. Generate the GRUB2 configuration file and reboot
```sh
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot now
```

7. Verify kernel driver is vfio-pci
```sh
lspci -nnv | grep -A10 -E 'VGA|Audio|NVMe'
```

8. Install virtualization tools (libvirt, qemu, etc)

```sh
sudo dnf group install -y --with-optional virtualization
sudo dnf install -y qemu-kvm-core libvirt guestfs-tools libguestfs-tools # extras for building, editing images
sudo dnf install -y edk2-ovmf swtpm swtpm-tools # for tpm, secureboot
sudo systemctl enable --now libvirtd
sudo virsh net-autostart default
sudo usermod -aG libvirt $LOGNAME

### optionally create a bridge network, so the VM has an IP from the LAN
### in examples here, we provide `--network bridge=br-enp113s0` flag in `virt-install` command
ETH_NIC=enp113s0 # retrieve from `nmcli device` output
sudo nmcli connection add type bridge ifname br-$ETH_NIC con-name br-$ETH_NIC
sudo nmcli connection add type ethernet ifname $ETH_NIC master br-$ETH_NIC
sudo nmcli connection up br-$ETH_NIC
sudo nmcli connection modify br-$ETH_NIC connection.autoconnect yes # set bridge to autoconnect
# this bridge setup with nmcli might require reboot to work properly
```

## Guest (any OS)
- ### Windows 11

```sh
sudo virt-install --name win11-pt-1 \
  --cpu host-passthrough,cache.mode=passthrough \
  --vcpus 16,maxvcpus=16,sockets=1,cores=8,threads=2 \
  --memory 32768 \
  --os-variant win11 \
  --graphics spice,gl=yes,listen=none \
  --video virtio,vram=262144,heads=2 \
  --console pty,target_type=serial \
  --noautoconsole \
  --serial pty \
  --network bridge=br-enp113s0 \
  --network bridge=virbr0 \
  --cdrom /var/lib/libvirt/boot/Win11_24H2_EnglishInternational_x64.iso \
  --disk /var/lib/libvirt/images/win11-pt-1.qcow2,size=500,bus=virtio \
  --boot uefi \
  --boot hd,cdrom,menu=on \
  --boot loader=/usr/share/edk2/ovmf/OVMF_CODE.secboot.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/edk2/ovmf/OVMF_VARS.secboot.fd,loader_secure=yes \
  --tpm emulator,model=tpm-crb,version=2.0 \
  --hostdev pci_0000_03_00_0 \
  --hostdev pci_0000_03_00_1 \
  --hostdev pci_0000_6e_00_0 \
  --import
```

- ### RHEL 9.4
See [extras.md](./extras.md) 
