This tutorial allows to create a KVM for gaming.

The VM is close to native performance with 3% of performance losses.

### **Table Of Contents**
- [**Thanks to**](#thanks-to)
- [**Enable & Verify IOMMU**](#enable-verify-iommu)
- [**Install required tools**](#install-required-tools)
- [**Enable required services**](#enable-required-services)
- [**Setup Guest OS**](#setup-guest-os)
- [**Install Windows**](#install-windows)
- [**Attaching PCI devices**](#attaching-pci-devices)
- [**Libvirt Hook Helper**](#libvirt-hook-helper)
- [**Config Libvirt Hooks**](#config-libvirt-hooks)
- [**Start/Stop Libvirt Hooks**](#startstop-libvirt-hooks)
- [**Keyboard/Mouse Passthrough**](#keyboardmouse-passthrough)
- [**Audio Passthrough**](#audio-passthrough)
- [**Video card driver virtualisation detection**](#video-card-driver-virtualisation-detection)
- [**vBIOS Patching**](#vbios-patching)
- [**CPU Pinning**](#cpu-pinning)
- [**Hyper-V Enlightenments**](#hyper-v-enlightenments)
- [**Disk Tuning**](#disk-tuning)
- [**Hugepages**](#hugepages)
- [**CPU Governor**](#cpu-governor)
- [**Windows drivers**](#windows-drivers)
- [**Optimize Windows**](#optimize-windows)

### **Thanks to**

**[Arch wiki](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)**  
The best way to learn how GPU passthrough is working.

**[bryansteiner](https://github.com/bryansteiner/gpu-passthrough-tutorial)**  
The best tutorial on GPU passthrough!

**[QaidVoid](https://github.com/QaidVoid/Complete-Single-GPU-Passthrough)**  
The best tutorial to use VFIO!

**[joeknock90](https://github.com/joeknock90/Single-GPU-Passthrough)**  
Really good tutorial on the NVIDIA GPU patch.

**[SomeOrdinaryGamers](https://www.youtube.com/watch?v=BUSrdUoedTo)**  
Bring me in the VFIO community.

**[Zeptic](https://www.youtube.com/watch?v=VKh2eKPnmXs)**  
How to get good performances in nested virtualization.

**[Quentin Franchi](https://gitlab.com/dev.quentinfranchi/vfio)**  
The scripts for AMD GPUs.

### **Enable & Verify IOMMU**

Ensure that ***AMD-Vi*** or ***Intel VT-d*** is supported by the CPU and enabled in the BIOS settings.

Enable IOMMU support by setting the kernel parameter depending on your CPU.

| /etc/default/grub                                              |
|----------------------------------------------------------------|
| `GRUB_CMDLINE_LINUX_DEFAULT="... amd_iommu=on iommu=pt ..."`   |
| OR                                                             |
| `GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on iommu=pt ..."` |

After rebooting, check that the groups are valid.
```sh
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

Example output: 
```
IOMMU Group 2:
    00:03.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge [1022:1482]
    00:03.1 PCI bridge [0604]: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge [1022:1483]
    09:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU104 [GeForce RTX 2070 SUPER] [10de:1e84] (rev a1)
    09:00.1 Audio device [0403]: NVIDIA Corporation TU104 HD Audio Controller [10de:10f8] (rev a1)
    09:00.2 USB controller [0c03]: NVIDIA Corporation TU104 USB 3.1 Host Controller [10de:1ad8] (rev a1)
    09:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU104 USB Type-C UCSI Controller [10de:1ad9] (rev a1)
```

If your card is not in an isolated group, you need to perform [ACS override patch](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch)).

### **Install required tools**

```sh
pacman -S --needed qemu libvirt edk2-ovmf virt-manager dnsmasq ebtables
```

### **Enable required services**

```sh
systemctl enable --now libvirtd
```

Start the default network manually.
```sh
virsh net-start default
virsh net-autostart default
```

### **Setup Guest OS**

Download [virtio](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) driver.

Create your storage volume with the ***raw*** format. Select ***Customize before install*** on Final Step. 

| In Overview                  |
|:-----------------------------|
| set **Chipset** to **Q35**   |
| set **Firmware** to **UEFI** |

| In CPUs                                              |
|:-----------------------------------------------------|
| set **CPU model** to **host-passthrough**            |
| set **CPU Topology** match your cpu topology -1 core |

| In Sata                        |
|:-------------------------------|
| set **Disk Bus** to **virtio** |

| In NIC                             |
|:-----------------------------------|
| set **Device Model** to **virtio** |

| In Add Hardware                                            |
|:-----------------------------------------------------------|
| select **CDROM** and point to `/path/to/virtio-driver.iso` |

### **Install Windows**

Windows can't detect the ***virtio disk***, so you need to ***Load Driver*** and select `virtio-iso/amd64/win10` when prompted.

Windows can't connect to the internet, we will activate internet later in this tutorial.

### **Attaching PCI devices**

The devices you want to passthrough.

| In Add PCI Host Device          |
|:--------------------------------|
| *PCI Host devices for your GPU* |
| *Audio Controller*              |

| In Add USB Host Device  |
|:------------------------|
| *Add whatever you want* |

| Remove          |
|:----------------|
| `Display spice` |
| `Channel spice` |
| `Video QXL`     |
| `Sound ich*`    |

### **Libvirt Hook Helper**

Libvirt hooks automate the process of running specific tasks during VM state change.

More documentation on [The Passthrough Post](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/) website.

<details>
  <summary><b>Create Libvirt Hook Helper</b></summary>

```sh
mkdir /etc/libvirt/hooks
nvim /etc/libvirt/hooks/qemu
chmod +x /etc/libvirt/hooks/qemu
```

  <table>
  <tr>
  <th>
  /etc/libvirt/hooks/qemu
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash
#
# Author: Sebastiaan Meijer (sebastiaan@passthroughpo.st)
#

GUEST_NAME="$1"
HOOK_NAME="$2"
STATE_NAME="$3"
MISC="${@:4}"

BASEDIR="$(dirname $0)"

HOOKPATH="$BASEDIR/qemu.d/$GUEST_NAME/$HOOK_NAME/$STATE_NAME"

set -e # If a script exits with an error, we should as well.

# check if it's a non-empty executable file
if [ -f "$HOOKPATH" ] && [ -s "$HOOKPATH"] && [ -x "$HOOKPATH" ]; then
    eval \"$HOOKPATH\" "$@"
elif [ -d "$HOOKPATH" ]; then
    while read file; do
        # check for null string
        if [ ! -z "$file" ]; then
          eval \"$file\" "$@"
        fi
    done <<< "$(find -L "$HOOKPATH" -maxdepth 1 -type f -executable -print;)"
fi
```

  </td>
  </tr>
  </table>
</details>

### **Config Libvirt Hooks**

This configuration file allows you to create variables that can be read by the scripts below.

```sh
nvim /etc/libvirt/hooks/kvm.conf
```

<table>
<tr>
<th>
/etc/libvirt/hooks/kvm.conf
</th>
</tr>

<tr>
<td>

```conf
# CONFIG
VM_MEMORY=13312

# VIRSH
VIRSH_GPU_VIDEO=pci_0000_09_00_0
VIRSH_GPU_AUDIO=pci_0000_09_00_1
VIRSH_USB=pci_0000_09_00_2
VIRSH_SERIAL_BUS=pci_0000_09_00_3
```

</td>
</tr>
</table>

`VM_MEMORY` in MiB is the memory allocated tho the guest.

Make sure to substitute the correct bus addresses for the devices you'd like to passthrough to your VM.
Just in case it's still unclear, you get the virsh PCI device IDs from the [Enable & Verify IOMMU](#enable-verify-iommu) script.
Translate the address for each device as follows: IOMMU `Group 1 01:00.0 ...` --> `VIRSH_...=pci_0000_01_00_0`.

### **Start/Stop Libvirt Hooks**

This command will set the variable KVM_NAME so you can execute the rest of the commands without changing the name of the VM.

```sh
KVM_NAME="YOUR_VM_NAME"
```

**If the scripts are not working, use the scripts as template and write your own.**

***Choose the Start/Stop scripts that most closely match your hardware.***

My hardware for this scripts is:
- *AMD Ryzen 7 3700X*
- *NVIDIA GeForce RTX 2070 SUPER*

<details>
  <summary><b>Create Start Script</b></summary>

```sh
mkdir -p /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/start.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/start.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/prepare/begin/start.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash
# Helpful to read output when debugging
set -x

# Load variables
source "/etc/libvirt/hooks/kvm.conf"

# Stop display manager
systemctl stop lightdm.service

# Unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Unbind EFI-Framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# Avoid a Race condition
sleep 5

# Unload all Nvidia drivers
modprobe -r nvidia_drm
modprobe -r nvidia_modeset
modprobe -r nvidia_uvm
modprobe -r nvidia

# Unbind the GPU from display driver
virsh nodedev-detach $VIRSH_GPU_VIDEO
virsh nodedev-detach $VIRSH_GPU_AUDIO
virsh nodedev-detach $VIRSH_USB
virsh nodedev-detach $VIRSH_SERIAL_BUS

# Load VFIO Kernel Module  
modprobe vfio-pci 
```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create Stop Script</b></summary>

```sh
mkdir -p /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/stop.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/stop.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/release/end/stop.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash
set -x

# Load variables
source "/etc/libvirt/hooks/kvm.conf"

# Unload VFIO-PCI Kernel Driver
modprobe -r vfio-pci
modprobe -r vfio_iommu_type1
modprobe -r vfio

# Re-Bind GPU to Nvidia Driver
virsh nodedev-reattach $VIRSH_GPU_VIDEO
virsh nodedev-reattach $VIRSH_GPU_AUDIO
virsh nodedev-reattach $VIRSH_USB
virsh nodedev-reattach $VIRSH_SERIAL_BUS

# Rebind VT consoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind

# Bind EFI-Framebuffer
nvidia-xconfig --query-gpu-info > /dev/null 2>&1
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

# Load all Nvidia drivers
modprobe nvidia_drm
modprobe nvidia_modeset
modprobe drm_kms_helper
modprobe drm
modprobe nvidia_uvm
modprobe nvidia

# Restart Display Manager
systemctl start lightdm.service
```

  </td>
  </tr>
  </table>
</details>

[Quentin](https://gitlab.com/dev.quentinfranchi/vfio) hardware for this scripts is:
- *AMD Ryzen 5 2600*
- *Radeon RX 590 Series*

<details>
  <summary><b>Create Start Script</b></summary>

```sh
mkdir -p /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/start.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/start.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/prepare/begin/start.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash
set -x

# load variables
source "/etc/libvirt/hooks/kvm.conf"

# Stop display manager
systemctl stop lightdm.service

# Stop pipewire
pulse_pid=$(pgrep -u quentin pulseaudio)
pipewire_pid=$(pgrep -u quentin pipewire-media)
kill $pulse_pid
kill $pipewire_pid

# Unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Unbind EFI Framebuffer
# echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# Avoid a race condition by waiting a couple of seconds. This can be calibrated to be shorter or longer if required for your system
sleep 5

# Unload AMD kernel module
modprobe -r amdgpu

# Detach GPU devices from host
virsh nodedev-detach $VIRSH_GPU_VIDEO
virsh nodedev-detach $VIRSH_GPU_AUDIO

# Load vfio module
modprobe vfio
modprobe vfio_pci
modprobe vfio_iommu_type1
```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create Stop Script</b></summary>

```sh
mkdir -p /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/stop.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/stop.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/release/end/stop.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash
set -x

# load variables
source "/etc/libvirt/hooks/kvm.conf"

# Unload all the vfio modules
modprobe -r vfio_pci
modprobe -r vfio_iommu_type1
modprobe -r vfio

# Attach GPU devices to host
# Use your GPU and HDMI Audio PCI host device
virsh nodedev-reattach $VIRSH_GPU_VIDEO
virsh nodedev-reattach $VIRSH_GPU_AUDIO

# Rebind VTconsoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind

# Rebind framebuffer to host
# echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

# Load AMD kernel module
modprobe  amdgpu
modprobe  gpu_sched
modprobe  ttm
modprobe  drm_kms_helper
modprobe  i2c_algo_bit
modprobe  drm
modprobe  snd_hda_intel

# Restart Display Manager
systemctl start lightdm.service
```

  </td>
  </tr>
  </table>
</details>

### **Keyboard/Mouse Passthrough**

Change the first line of the xml to:

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
```

</td>
</tr>
</table>

Find your keyboard and mouse devices in ***/dev/input/by-id***. You'd generally use the devices ending with ***event-kbd*** and ***event-mouse***. And the devices in your configuration right before closing `</domain>` tag.

You can verify if it works by `cat /dev/input/by-id/DEVICE_NAME`.

Replace ***MOUSE_NAME*** and ***KEYBOARD_NAME*** with your device id.

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <qemu:commandline>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/MOUSE_NAME'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/KEYBOARD_NAME,grab_all=on,repeat=on'/>
  </qemu:commandline>
</domain>
```

</td>
</tr>
</table>

You need to include these devices in your qemu config.

<table>
<tr>
<th>
/etc/libvirt/qemu.conf
</th>
</tr>

<tr>
<td>

```conf
...
user = "YOUR_USERNAME"
group = "kvm"
...
cgroup_device_acl = [
    "/dev/input/by-id/KEYBOARD_NAME",
    "/dev/input/by-id/MOUSE_NAME",
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
    "/dev/rtc","/dev/hpet", "/dev/sev"
]
...
```

</td>
</tr>
</table>

Also, add the virtio devices (You cannot remove the PS/2 devices).

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
<devices>
  ...
  <input type='mouse' bus='virtio'/>
  <input type='keyboard' bus='virtio'/>
  ...
</devices>
...
```

</td>
</tr>
</table>

### **Audio Passthrough**

VM's audio can be routed to the host. You need **Pulseaudio**.

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <qemu:commandline>
    ...
    <qemu:arg value="-device"/>
    <qemu:arg value="ich9-intel-hda,bus=pcie.0,addr=0x1b"/>
    <qemu:arg value="-device"/>
    <qemu:arg value="hda-micro,audiodev=hda"/>
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=/run/user/1000/pulse/native"/>
  </qemu:commandline>
</devices>
```

</td>
</tr>
</table>

### **Video card driver virtualisation detection**

Video Card drivers refuse to run in Virtual Machine, so you need to spoof Hyper-V Vendor ID.

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
<features>
  ...
  <hyperv>
    ...
    <vendor_id state='on' value='buttplug'/>
    ...
  </hyperv>
  ...
</features>
...
```

</td>
</tr>
</table>

NVIDIA guest drivers also require hiding the KVM CPU leaf:

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
<features>
  ...
  <kvm>
    <hidden state='on'/>
  </kvm>
  <ioapic driver="kvm"/>
  ...
</features>
...
```

</td>
</tr>
</table>

### **vBIOS Patching**

<details>
  <summary><b>How to patch NVIDIA vBIOS</b></summary>
  
  **Only NVIDIA GPU's need to be patched**

  To get a rom for your GPU you can either download one from [here](https://www.techpowerup.com/vgabios/) or use nvflash to dump the bios currently on your GPU.

  Use the dumped/downloaded vbios and open it in a hex editor.

  Search for the strings "VIDEO".
  ![images/vbios1.jpg](images/vbios1.jpg)

  Then you have to search for the first U that is in front of VIDEO.
  ![images/vbios2.jpg](images/vbios2.jpg)

  Delete all of the code above the U then save your patched vbios.

</details>

To add the patched rom, in ***hostdev*** add ***rom***, only for the VGA pci:

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    ...
  </source>
  <rom file="/path/to/patched-vbios.rom"/>
  ...
</hostdev>
...
```

</td>
</tr>
</table>

### **CPU Pinning**

My setup is an AMD Ryzen 7 3700X which has 8 physical cores and 16 threads (2 threads per core).

<details>
  <summary><b>How to bind the threads to the core</b></summary>

It's very important that when we passthrough a core, we include its sibling. To get a sense of your cpu topology, use the command `lscpu -e`. A matching core id (i.e. "CORE" column) means that the associated threads (i.e. "CPU" column) run on the same physical core.

```
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE    MAXMHZ    MINMHZ
  0    0      0    0 0:0:0:0          yes 4823.4370 2200.0000
  1    0      0    1 1:1:1:0          yes 4559.7651 2200.0000
  2    0      0    2 2:2:2:0          yes 4689.8428 2200.0000
  3    0      0    3 3:3:3:0          yes 4426.1709 2200.0000
  4    0      0    4 4:4:4:1          yes 5224.2178 2200.0000
  5    0      0    5 5:5:5:1          yes 5090.6250 2200.0000
  6    0      0    6 6:6:6:1          yes 5224.2178 2200.0000
  7    0      0    7 7:7:7:1          yes 4957.0308 2200.0000
  8    0      0    0 0:0:0:0          yes 4823.4370 2200.0000
  9    0      0    1 1:1:1:0          yes 4559.7651 2200.0000
 10    0      0    2 2:2:2:0          yes 4689.8428 2200.0000
 11    0      0    3 3:3:3:0          yes 4426.1709 2200.0000
 12    0      0    4 4:4:4:1          yes 5224.2178 2200.0000
 13    0      0    5 5:5:5:1          yes 5090.6250 2200.0000
 14    0      0    6 6:6:6:1          yes 5224.2178 2200.0000
 15    0      0    7 7:7:7:1          yes 4957.0308 2200.0000
```

According to the logic seen above, here are my core and their threads binding.

```
Core 1: 0, 8
Core 2: 1, 9
Core 3: 2, 10
Core 4: 3, 11
Core 5: 4, 12
Core 6: 5, 13
Core 7: 6, 14
Core 8: 7, 15
```

</details>

In this example, I want to get 1 core for the host and 7 cores for the guest. 
I will let the ***core 1*** for my host, so ***0*** and ***8*** are the logical threads.

I show you the final result, everything will be explained below.

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <vcpu placement="static">14</vcpu>
  <iothreads>1</iothreads>
  <cputune>
    <vcpupin vcpu="0" cpuset="1"/>
    <vcpupin vcpu="1" cpuset="9"/>
    <vcpupin vcpu="2" cpuset="2"/>
    <vcpupin vcpu="3" cpuset="10"/>
    <vcpupin vcpu="4" cpuset="3"/>
    <vcpupin vcpu="5" cpuset="11"/>
    <vcpupin vcpu="6" cpuset="4"/>
    <vcpupin vcpu="7" cpuset="12"/>
    <vcpupin vcpu="8" cpuset="5"/>
    <vcpupin vcpu="9" cpuset="13"/>
    <vcpupin vcpu="10" cpuset="6"/>
    <vcpupin vcpu="11" cpuset="14"/>
    <vcpupin vcpu="12" cpuset="7"/>
    <vcpupin vcpu="13" cpuset="15"/>
    <emulatorpin cpuset="0,8"/>
    <iothreadpin iothread="1" cpuset="0,8"/>
  </cputune>
  ...
</domain>
```

</td>
</tr>
</table>

<details>
  <summary><b>Explanations of cpu pinning</b></summary>

  <table>
  <tr>
  <td>
  Number of threads to passthrough
  </td>
  </tr>

  <tr>
  <td>

```xml
<vcpu placement="static">14</vcpu>
```

  </td>
  </tr>
  </table>

  <table>
  <tr>
  <td>
  Same number as the iothreadpin below
  </td>
  </tr>

  <tr>
  <td>

```xml
<iothreads>1</iothreads>
```

  </td>
  </tr>
  </table>

  <table>
  <tr>
  <td>
  cpuset corresponds to the bindings of your host core
  </td>
  </tr>

  <tr>
  <td>

```xml
<cputune>
  ...
  <emulatorpin cpuset="0,8"/>
  <iothreadpin iothread="1" cpuset="0,8"/>
</cputune>
```

  </td>
  </tr>
  </table>

  <table>
  <tr>
  <td>
  vcpu corresponds to the guest cores, increment by 1 starting with 0.

  cpuset correspond to your threads you want to passthrough. It is necessary that your core and their threads binding follow each other.
  </td>
  </tr>

  <tr>
  <td>

```xml
<cputune>
  <vcpupin vcpu="0" cpuset="1"/>
  <vcpupin vcpu="1" cpuset="9"/>
  <vcpupin vcpu="2" cpuset="2"/>
  <vcpupin vcpu="3" cpuset="10"/>
  <vcpupin vcpu="4" cpuset="3"/>
  <vcpupin vcpu="5" cpuset="11"/>
  <vcpupin vcpu="6" cpuset="4"/>
  <vcpupin vcpu="7" cpuset="12"/>
  <vcpupin vcpu="8" cpuset="5"/>
  <vcpupin vcpu="9" cpuset="13"/>
  <vcpupin vcpu="10" cpuset="6"/>
  <vcpupin vcpu="11" cpuset="14"/>
  <vcpupin vcpu="12" cpuset="7"/>
  <vcpupin vcpu="13" cpuset="15"/>
  ...
</cputune>
```

  </td>
  </tr>
  </table>
</details>

You need to match your CPU pathrough.

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="7" threads="2"/>
    <cache mode="passthrough"/>
    <feature policy="require" name="topoext"/>
  </cpu>
  ...
</domain>
```

</td>
</tr>
</table>

### **Hyper-V Enlightenments**

Hyper-V enlightenments help the guest VM handle virtualization tasks.

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <qemu:commandline>
    ...
    <qemu:arg value="-rtc"/>
    <qemu:arg value="base=localtime"/>
    <qemu:arg value="-cpu"/>
    <qemu:arg value="host,host-cache-info=on,kvm=off,l3-cache=on,kvm-hint-dedicated=on,migratable=no,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time,hv_vendor_id=buttplug,+invtsc,+topoext"/>
  </qemu:commandline>
</devices>
```

</td>
</tr>
</table>

<details>
  <summary><b>You can alternatively use this config</b></summary>

  I do not use this configuration because I experienced mouse latency in games.

  <table>
  <tr>
  <th>
  XML
  </th>
  </tr>

  <tr>
  <td>

  ```xml
  <features>
      ...
      <hyperv>
        ...
        <vpindex state='on'/>
        <synic state='on'/>
        <stimer state='on'/>
        <reset state='on'/>
        <frequencies state='on'/>
      </hyperv>
      ...
  </features>
  ```

  </td>
  </tr>
  </table>
</details>

### **Disk Tuning**

KVM and QEMU provide two paravirtualized storage backends:
- virtio-blk (default)
- virtio-scsi (new)

For virtio-blk, you need to replace the `driver` line by:

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

You have to make `queues` correspond to the number of ***vcpus*** you pass to the host. In my case ***14*** because I pass *7* cores with *2* threads per core. Remember the [CPU Pinning](#cpu-pinning) section.

```xml
...
<devices>
  ...
  <disk type="file" device="disk">
    <driver name="qemu" type="raw" cache="none" io="threads" discard="unmap" iothread="1" queues="14"/>
    ...
  </disk>
  ...
</devices>
...
```

</td>
</tr>
</table>

For virtio-scsi, follow [bryansteiner](https://github.com/bryansteiner/gpu-passthrough-tutorial/#----disk-tuning) tutorial.

### **Hugepages**

Memory (RAM) is divided up into basic segments called pages. By default, the x86 architecture has a page size of 4KB. CPUs utilize pages within the built in memory management unit ([MMU](https://en.wikipedia.org/wiki/Memory_management_unit)). Although the standard page size is suitable for many tasks, hugepages are a mechanism that allow the Linux kernel to take advantage of large amounts of memory with reduced overhead. Hugepages can vary in size anywhere from 2MB to 1GB.

Many tutorials will have you reserve hugepages for your guest VM at host boot-time. There's a significant downside to this approach: a portion of RAM will be unavailable to your host even when the VM is inactive. In [bryansteiner](https://github.com/bryansteiner/gpu-passthrough-tutorial) setup, he chose to allocate hugepages before the VM starts and deallocate those pages on VM shutdown.

<details>
  <summary><b>Create Alloc Hugepages Script</b></summary>

```sh
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/alloc_hugepages.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/alloc_hugepages.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/prepare/begin/alloc_hugepages.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

## Load the config file
source "/etc/libvirt/hooks/kvm.conf"

## Calculate number of hugepages to allocate from memory (in MB)
HUGEPAGES="$(($VM_MEMORY/$(($(grep Hugepagesize /proc/meminfo | awk '{print $2}')/1024))))"

echo "Allocating hugepages..."
echo $HUGEPAGES > /proc/sys/vm/nr_hugepages
ALLOC_PAGES=$(cat /proc/sys/vm/nr_hugepages)

TRIES=0
while (( $ALLOC_PAGES != $HUGEPAGES && $TRIES < 1000 ))
do
  echo 1 > /proc/sys/vm/compact_memory            ## defrag ram
  echo $HUGEPAGES > /proc/sys/vm/nr_hugepages
  ALLOC_PAGES=$(cat /proc/sys/vm/nr_hugepages)
  echo "Succesfully allocated $ALLOC_PAGES / $HUGEPAGES"
  let TRIES+=1
done

if [ "$ALLOC_PAGES" -ne "$HUGEPAGES" ]
then
  echo "Not able to allocate all hugepages. Reverting..."
  echo 0 > /proc/sys/vm/nr_hugepages
  exit 1
fi
```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create Dealloc Hugepages Script</b></summary>

```sh
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/dealloc_hugepages.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/dealloc_hugepages.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/release/end/dealloc_hugepages.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

echo 0 > /proc/sys/vm/nr_hugepages
```

  </td>
  </tr>
  </table>
</details>

<table>
<tr>
<th>
XML
</th>
</tr>

<tr>
<td>

```xml
...
  <memory unit="KiB">13631488</memory>
  <currentMemory unit="KiB">13631488</currentMemory>
  <memoryBacking>
    <hugepages/>
  </memoryBacking>
  ...
</domain>

```

</td>
</tr>
</table>

The memory need to match your `VM_MEMORY` from your config *(to convert KiB to MiB you need to divide by 1024)*.

### **CPU Governor**

This performance tweak takes advantage of the [CPU frequency scaling governor](https://wiki.archlinux.org/title/CPU_frequency_scaling#Scaling_governors) in Linux.

<details>
  <summary><b>Create CPU Performance Script</b></summary>

```sh
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/cpu_mode_performance.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/prepare/begin/cpu_mode_performance.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/prepare/begin/cpu_mode_performance.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

## Enable CPU governor performance mode
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
for file in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo "performance" > $file; done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Create CPU Ondemand Script</b></summary>

```sh
nvim /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/cpu_mode_ondemand.sh
chmod +x /etc/libvirt/hooks/qemu.d/$KVM_NAME/release/end/cpu_mode_ondemand.sh
```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/VM_NAME/release/end/cpu_mode_ondemand.sh
  </th>
  </tr>

  <tr>
  <td>

```sh
#!/bin/bash

## Enable CPU governor on-demand mode
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
for file in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo "ondemand" > $file; done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

  </td>
  </tr>
  </table>
</details>

### **Windows drivers**

To get the *network*, *sound*, *mouse* and *keyboard* working properly you need to install the drivers.

In `Device Manager` update *network*, *sound*, *mouse* and *keyboard* drivers with the local virtio iso `/path/to/virtio-driver`.

### **Optimize Windows**

#### *Windows debloater*

```powershell
iwr -useb https://git.io/debloat|iex
```

#### *Better performances*

In *Windows Settings*:
- set ***Power suply*** to ***Performances***

If you have and NVIDIA card, in *NVIDIA Control Panel*:
- set ***Texture filtering quality*** to ***High performance***
- set ***Power management mode*** to ***Max performance***

#### *Disable Windows Defender*

Video tutorial: [How to Completely Turn Off Windows Defender in Windows 10](https://youtu.be/31TDHRegTLM)

```
Windows security & threat protection (
    Deactivate all buttons
)
```
```
Task Scheduler (
    Task Scheduler Library
        Microsoft
            Windows
                Windows Defender (
                    Select all
                    Right click
                    Disable
                )
)
```
```
Edit Group Policy (
    Computer Configuration
        Administrative Templates
            Windows Components
                Microsoft Defender Antivirus (
                    Turn off Microsoft Defender Antivirus
                    Select Enabled
                )
)
```