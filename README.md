# GPU passthrough setup

GPU passthrough enables a virtual machine (VM) of any OS to have its dedicated GPU with near-native performance. See [PCI_passthrough_via_OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) for more.

So far I've got

- pytorch detect GPU (ie CUDA through ROCm) on a RHEL9.4 VM
- [AMD Adrenalin](https://www.amd.com/en/products/software/adrenalin.html) detect GPU on a Windows11 VM

But still having **Display output is not active** rendering issue ie can't game on Windows VM;
that's why documenting my progress to seek help as well as help whoever interested.

## Host (Linux)
Tested on Fedora41, Ryzen 7950X, RX 7900XTX, X870E. Adjustments needed for Intel CPU and Nvidia GPU.

1. Enable Virtualization and IOMMU on BIOS/UEFI. Then ensure Virtualization and IOMMU from host shell.
```sh
sudo dmesg | grep -i -e DMAR -e IOMMU

shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

2. Find the [BDF](https://wiki.xenproject.org/wiki/Bus:Device.Function_(BDF)_Notation) id, `vendor_id:device_id` of the GPU to be passed-through

```sh
lspci -nnv | grep -iE -A10 'navi|nvidia'
```

Expected output should look like

```sh
03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 [Radeon RX 7900 XT/7900 XTX/7900 GRE/7900M] [1002:744c] (rev c8) (prog-if 00 [VGA controller])
	Subsystem: Sapphire Technology Limited NITRO+ RX 7900 XTX Vapor-X [1da2:e471]
	Flags: bus master, fast devsel, latency 0, IRQ 175, IOMMU group 15
	Kernel driver in use: amdgpu
	Kernel modules: amdgpu
  # more stuff truncated

03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
	Flags: bus master, fast devsel, latency 0, IRQ 194, IOMMU group 16
	Kernel driver in use: snd_hda_intel
	Kernel modules: snd_hda_intel
  # more stuff truncated
```

> **On host we need to ensure `Kernel driver in use: vfio-pci` (as opposed to `amdgpu` / `nvidia` etc) for the devices**

<p align="center">
  <img src="./vfio-pcie-passthrough.mmd.svg" alt="description" height="500"/>
</p>

GPUs typically have a VGA and an Audio *Function* / component. From `lspci` output above

Bus | Device | Function| vendor_id:device_id| vfio-pci.ids        | hostdev (derived) |
----|--------|---------|--------------------|---------------------|-------------------|
03  |   00   |    0    |    1002:744c       | 1002:744c,1002:ab30 | pci_0000_03_00_0  |
03  |   00   |    1    |    1002:ab30       |                     | pci_0000_03_00_1  |

We use the identifiers
- **vfio-pci.ids** to prevent default kernel driver from taking control of the target GPU so vfio-pci has exclusive control of it
- **hostdev** to attach the GPU to any VM; a GPU can't be attached to more than one **running** VM

3. Update `GRUB_CMDLINE_LINUX` in `/etc/default/grub`
```sh
# truncated
GRUB_CMDLINE_LINUX="APPEND TO YOUR EXISTING CONFIG rd.driver.blacklist=amdgpu modprobe.blacklist=amdgpu video=efifb:off amd_iommu=on amd_iommu=pt rd.driver.pre=vfio-pci kvm.ignore_msrs=1 vfio-pci.ids=1002:744c,1002:ab30"
# truncated
```

4. Configure VFIO for device passthrough and blacklist conflicting drivers
```sh
cat <<EOF | sudo tee /etc/modprobe.d/vfio.conf 
options vfio-pci ids=1002:744c,1002:ab30
options vfio_iommu_type1 allow_unsafe_interrupts=1
softdep drm pre: vfio-pci
EOF

cat <<EOF | sudo tee /etc/modprobe.d/vfio-blacklist.conf 
blacklist amdgpu
blacklist snd_hda_intel
EOF
```

5. Configure `dracut` to include VFIO drivers in initramfs
```sh
cat <<EOF | sudo tee /etc/dracut.conf.d/00-vfio.conf 
force_drivers+=" vfio_pci vfio vfio_iommu_type1 "
EOF
```

6. Regenerate initramfs with VFIO drivers
```sh
sudo dracut -f --kver $(uname -r)
```

7. Generate the GRUB2 configuration file
```sh
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot now
```

8. Verify kernel driver is vfio-pci
```sh
spci -nnv | grep -iE -A10 'navi|nvidia'
```

expected output
```sh
03:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 [Radeon RX 7900 XT/7900 XTX/7900 GRE/7900M] [1002:744c] (rev c8)
	Subsystem: Sapphire Technology Limited NITRO+ RX 7900 XTX Vapor-X [1da2:e471]
	Kernel driver in use: vfio-pci   <-------------------------------------------
	Kernel modules: amdgpu
03:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Navi 31 HDMI/DP Audio [1002:ab30]
	Kernel driver in use: vfio-pci   <-------------------------------------------
	Kernel modules: snd_hda_intel
```

9. Install virtualization tools (libvirt, qemu, etc)
```sh
sudo dnf group install -y --with-optional virtualization
sudo dnf install -y qemu-kvm-core libvirt guestfs-tools guestfish libguestfs-tools # extras for building, editing images
sudo dnf install -y edk2-ovmf swtpm swtpm-tools # for tpm, secureboot
sudo systemctl enable --now libvirtd
sudo virsh net-autostart default

### optionally create a bridge a network, so the VM has an IP from the LAN
### in examples here, virt-install provide `--network bridge=br-enp113s0` flag
ETH_NIC=enp113s0 # retrieve from `nmcli device` output
sudo nmcli connection add type bridge ifname br-$ETH_NIC con-name br-$ETH_NIC
sudo nmcli connection add type ethernet ifname $ETH_NIC master br-$ETH_NIC
sudo nmcli connection up br-$ETH_NIC
sudo nmcli connection modify br-$ETH_NIC connection.autoconnect yes # set bridge to autoconnect
# this bridge setup with nmcli might require reboot to work properly
```

## Guest (any OS)

### Linux
Tested passthrough on RHEL9.4 with AMD ROCm 6.2. I couldn't yet get ROCm to work (reliably, consistently) on Ubuntu24.04.

> **`virt-install`** command occasionally errors
```sh
ERROR    internal error: Could not run '/usr/bin/swtpm_setup'. exitstatus: 1; Check error log '/var/log/swtpm/libvirt/qemu/ubuntu2404-rocm6-2-swtpm.log' for details.
Domain installation does not appear to have been successful.
```

Just rerun the commnad. That's the fix for now.

1. Download installation image
- RHEL: Download from [https://access.redhat.com/downloads](https://access.redhat.com/downloads)

- Ubuntu
```sh
sudo curl -L https://releases.ubuntu.com/noble/ubuntu-24.04.1-live-server-amd64.iso -o /var/lib/libvirt/boot/ubuntu-24.04.1.iso
```

2. Install VM

- RHEL
```sh
sudo virt-install --name rhel94-rocm6 \
  --cpu host-passthrough,cache.mode=passthrough \
  --vcpus 16,maxvcpus=16,sockets=1,cores=8,threads=2 \
  --memory 32768 \
  --os-variant rhel9.4 \
  --graphics vnc  \
  --console pty,target_type=serial \
  --noautoconsole \
  --serial pty \
  --cdrom /var/lib/libvirt/boot/rhel-9.4-x86_64-dvd.iso \
  --disk /var/lib/libvirt/images/rhel94-rocm6.qcow2,size=500,bus=virtio \
  --network bridge=br-enp113s0 \
  --boot uefi \
  --boot hd,cdrom,menu=on \
  --boot loader=/usr/share/edk2/ovmf/OVMF_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/edk2/ovmf/OVMF_VARS.fd \
  --hostdev pci_0000_03_00_0 --hostdev pci_0000_03_00_1
```

- Ubuntu
```sh
sudo virt-install --name ubuntu24-rocm6 \
  --cpu host-passthrough,cache.mode=passthrough \
  --vcpus 16,maxvcpus=16,sockets=1,cores=8,threads=2 \
  --memory 32768 \
  --os-variant ubuntu24.04 \
  --graphics vnc  \
  --console pty,target_type=serial \
  --noautoconsole \
  --serial pty \
  --cdrom /var/lib/libvirt/boot/ubuntu-24.04.1.iso \
  --disk /var/lib/libvirt/images/ubuntu24-rocm6.qcow2,size=500,bus=virtio \
  --network bridge=br-enp113s0 \
  --boot uefi \
  --boot hd,cdrom,menu=on \
  --boot loader=/usr/share/edk2/ovmf/OVMF_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/edk2/ovmf/OVMF_VARS.fd \
  --hostdev pci_0000_03_00_0 --hostdev pci_0000_03_00_1
```

Follow through a few steps in virt-manager GUI. Enable SSH server if you want to access VM shell from host or elsewhere.

3. Access the VM eg via SSH from host

```sh
# One way to find new VM's IP when bridge network is used with DHCP
nmap -sn 192.168.0.0/24 # replace with your (home LAN) subnet
# it prints discovered IPs, hopefully including that of newly created VM
# check if port 22 open of suspected IP
nc -zv 192.168.0.221 22 # replace
## optionally copy ssh key
# ssh-copy-id username_entered_during_installation@VM_IP
# SSH into the VM
ssh username_entered_during_installation@VM_IP
```

You might want to see [ide.md](./ide.md) if you use text-based editor like vim over SSH.

4. Install AMD ROCm

- Ubuntu
```sh
sudo apt update
sudo apt install -y "linux-headers-$(uname -r)" "linux-modules-extra-$(uname -r)"
sudo usermod -a -G render,video $LOGNAME # Add the current user to the render and video groups
wget https://repo.radeon.com/amdgpu-install/6.2.4/ubuntu/noble/amdgpu-install_6.2.60204-1_all.deb
sudo apt install -y ./amdgpu-install_6.2.60204-1_all.deb
sudo apt update
sudo apt install -y amdgpu-dkms rocm
sudo reboot now
```

- RHEL9
```sh
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo rpm -ivh epel-release-latest-9.noarch.rpm
sudo dnf install -y dnf-plugin-config-manager
sudo crb enable
sudo dnf install -y "kernel-headers-$(uname -r)" "kernel-devel-$(uname -r)"
sudo usermod -a -G render,video $LOGNAME # Add the current user to the render and video groups
sudo dnf install -y https://repo.radeon.com/amdgpu-install/6.2.4/rhel/9.4/amdgpu-install-6.2.60204-1.el9.noarch.rpm
sudo dnf clean all
sudo dnf install -y amdgpu-dkms rocm
sudo reboot now
```

5. Configure the system linker by indicating where to find the shared objects (.so files) for the ROCm applications
```sh
sudo tee --append /etc/ld.so.conf.d/rocm.conf <<EOF
/opt/rocm/lib
/opt/rocm/lib64
EOF
sudo ldconfig
````

6. Optionally verify pytorch can detect CUDA
```sh
# install python dev tools
## rhel
sudo dnf install -y libjpeg-devel python3-devel python3-pip
pip3 install wheel setuptools
## ubuntu
# sudo apt install -y libjpeg-dev python3-dev python3-pip python3-wheel python3-setuptools python3-venv

# create a python3 virtual environment
python3 -m venv ~/rocm
source ~/rocm/bin/activate
pip3 install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/rocm6.2/

python3 -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.device_count()); print(torch.cuda.get_device_name(0))"
### If everything went well, output should look like
# True
# 1
# Radeon RX 7900 XTX
```

### Windows
1. Download Windows11 ISO from https://www.microsoft.com/en-us/software-download/windows11. I've saved it at `/var/lib/libvirt/boot/Win11_24H2_EnglishInternational_x64.iso`

2. Download virtio-win driver. See [virtio-win/
kvm-guest-drivers-windows](https://github.com/virtio-win/kvm-guest-drivers-windows/wiki/Driver-installation) if you wanna learn more about virtio-win.

```sh
sudo curl -L https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/virtio-win.iso -o /var/lib/libvirt/boot/virtio-win.iso
```

3. Install Windows11 VM

```sh
sudo virt-install --name win11-1 \
  --cpu host-passthrough,cache.mode=passthrough \
  --vcpus 16,maxvcpus=16,sockets=1,cores=8,threads=2 \
  --memory 32768 \
  --os-variant win11 \
  --graphics spice \
  --video virtio \
  --console pty,target_type=serial \
  --noautoconsole \
  --serial pty \
  --cdrom /var/lib/libvirt/boot/Win11_24H2_EnglishInternational_x64.iso \
  --disk /var/lib/libvirt/images/win11-1.qcow2,size=500,bus=virtio \
  --disk path=/var/lib/libvirt/boot/virtio-win.iso,device=cdrom \
  --network bridge=br-enp113s0 \
  --boot uefi \
  --boot hd,cdrom,menu=on \
  --boot loader=/usr/share/edk2/ovmf/OVMF_CODE.secboot.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/edk2/ovmf/OVMF_VARS.secboot.fd,loader_secure=yes \
  --tpm emulator,model=tpm-crb,version=2.0 \
  --hostdev pci_0000_03_00_0 --hostdev pci_0000_03_00_1
```

Click through the wizards typical in Windows installation. Windows device driver can't detect vitio storage.
On the disk selection wizard click on `Load Driver` and point to virtio-win driver directory from mounted CDROM eg `E:\amd64\w11`. 

4. Install [AMD Adrenalin](https://www.amd.com/en/products/software/adrenalin.html) and verify it detects GPU.
