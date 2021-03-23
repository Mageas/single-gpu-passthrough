# VFIO Single GUP Passthrough

<!-- TOC -->

- [VFIO Single GUP Passthrough](#vfio-single-gup-passthrough)
    - [Enable & Verify IOMMU](#enable--verify-iommu)
    - [Install required tools](#install-required-tools)
    - [Setup Guest OS](#setup-guest-os)
    - [Guest OS Hardware](#guest-os-hardware)
        - [Add](#add)
        - [Remove Optional](#remove-optional)
    - [GPU patching](#gpu-patching)
    - [Passthrough the GPU](#passthrough-the-gpu)
        - [NVIDIA](#nvidia)
        - [AMD](#amd)
    - [Libvirt Hooks](#libvirt-hooks)
        - [NVIDIA](#nvidia)
        - [AMD](#amd)
    - [Optional customization](#optional-customization)
        - [CPU Pinning](#cpu-pinning)
        - [Nested virtualization](#nested-virtualization)

<!-- /TOC -->

## Enable & Verify IOMMU

Ensure that AMD-Vi is supported by the CPU and enabled in the BIOS settings.

Enable IOMMU support by setting the kernel parameter.

| /etc/default/grub                                          |
| :--------------------------------------------------------: |
| GRUB_CMDLINE_LINUX_DEFAULT="... amd_iommu=on iommu=pt ..." |

After rebooting, check that the groups are valid.
```
shopt -s nullglob
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

## Install required tools

TODO: Install tools.
```
pacman -S --needed qemu libvirt edk2-ovmf virt-manager dnsmasq ebtables
```

TODO: Enable libvirtd.
```
systemctl enable --now libvirtd
```

Configuring libvirt.
```
virsh net-start default
virsh net-autostart default
```

TODO: Add groups to user.
```
usermod -aG kvm,input,libvirt <username>
```

## Setup Guest OS

When the VM creation wizard asks you to name your VM (final step before clicking "Finish"), check the "Customize before install" checkbox.

**Overview**
- **Chipset** to *Q35*
- **Firmware** to *UEFI*

**CPUs**
- **CPU model** to *host-passthrough*
- **CPU Topology**:
    - **Sockets** to *1*
    - **Cores** to *(number of cores you want to passthrough)*
    - **Threads** to *(how many threads per core)*

You need to install the virtual machine.

## Guest OS Hardware

### Add

You need to passthrough hardware to the guest

In the *PCI Host Device* you have to add all the components of the graphics card (NVIDIA or AMD). You can optionally add your audio, etc.

In the *USB Host Device* you can add your mouse, keyboard, microphone, etc.

### Remove (Optional)

You can remove unused hardware 
- *Display spice*
- *Channel spice*
- *Video QXL*

## GPU patching

## Passthrough the GPU

### NVIDIA

Add custom rom to all **NVIDIA** components
```
<hostdev>
  ...
  <rom file="/path/to/patched-vbios.rom"/>
</hostdev>
```

You have to add these lines to avoid NVIDIA error 43
```
<features>
  ...
  <hyperv>
    ...
    <vendor_id state="on" value="buttplug"/>
  </hyperv>
  <kvm>
    <hidden state="on"/>
  </kvm>
</features>
```

### AMD

## Libvirt Hooks

### NVIDIA

### AMD

## Optional customization

### CPU Pinning

### Nested virtualization


































