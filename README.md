# GPD Duo macOS Sequoia on CachyOS KVM

Notes and configuration templates for a working macOS Sequoia VM on a GPD Duo
running CachyOS, with an AMD Radeon RX 6600M in an external GPU enclosure
passed through to the guest.

This is a field guide for a specific, tested configuration. It is not an
installer, does not include macOS, recovery media, OpenCore, firmware, or
Apple keys, and deliberately omits machine-specific identifiers.

## Tested configuration

| Component | Configuration |
| --- | --- |
| Host | GPD Duo with AMD Ryzen AI 9 HX 370 / Radeon 890M |
| Host OS | CachyOS |
| Hypervisor | libvirt system instance with QEMU/KVM 11.1 |
| Guest | macOS Sequoia |
| Firmware | OVMF: `OVMF_CODE.4m.fd` + a per-VM writable VARS file |
| Guest chipset | Q35 |
| Guest CPU | 12 vCPU, 1 socket x 12 cores x 1 thread |
| Guest memory | 32 GiB |
| GPU | AMD Navi 23 / Radeon RX 6600M, plus its HDMI/DP-audio function |
| Network | libvirt `default` NAT network, VMXNET3 |
| Input | Direct USB passthrough for a wired Razer mouse and Dell keyboard |

## Scope and important limits

- Check Apple's current macOS license terms before using macOS in a virtual
  machine or on non-Apple hardware.
- Do not publish an Apple SMC `osk` value, SMBIOS serials, UUIDs, MAC
  addresses, generated ROM values, recovery images, or EFI binaries.
- This guide does **not** pass through the host Radeon 890M, the host boot
  drive, or user data partitions.
- Back up the host and guest before editing firmware, PCI, or storage
  configuration.

The macOS/OpenCore baseline used here was derived from
[OSX-PROXMOX](https://github.com/luchina-gabriel/OSX-PROXMOX). This repository
contains original documentation and templates only; it does not redistribute
that project's files.

## Host prerequisites

1. Enable AMD IOMMU in firmware and the host kernel.
2. Confirm that the eGPU graphics and audio functions are isolated from the
   host, then bind both to `vfio-pci`.
3. Keep the GPD Duo's integrated Radeon 890M bound to the host `amdgpu`
   driver. It is not the passthrough GPU.
4. Confirm that the VM is shut off before adding the PCI devices.

Useful inspection commands:

```fish
for dev in 0000:65:00.0 0000:65:00.1
    echo "=== $dev ==="
    lspci -nnk -s $dev
    readlink -f /sys/bus/pci/devices/$dev/driver
end
```

For this machine, `65:00.0` is the Navi 23 graphics function and `65:00.1` is
its audio function. Replace those addresses with the values from your own
machine; PCI addresses are not portable.

## VM baseline

Use the system libvirt connection (`qemu:///system`) and Q35/OVMF. On CachyOS,
the tested firmware paths were:

```text
/usr/share/edk2/x64/OVMF_CODE.4m.fd
/usr/share/edk2/x64/OVMF_VARS.4m.fd
```

Create a distinct writable NVRAM file per VM; never write to the system VARS
template. The domain uses a copied file at a path similar to:

```text
/var/lib/libvirt/qemu/nvram/macos-sequoia_VARS.fd
```

Keep OpenCore as the first boot device. The recovery image is only required for
installation and recovery. The installed macOS disk in this setup is a qcow2
image on a virtio-blk controller with `cache='none'` and `discard='unmap'`.

## CPU and memory

The important post-install topology is one virtual socket with twelve cores,
not twelve one-core sockets:

```xml
<memory unit='KiB'>33554432</memory>
<currentMemory unit='KiB'>33554432</currentMemory>
<vcpu placement='static'>12</vcpu>
<cpu mode='host-passthrough' check='none' migratable='on'>
  <topology sockets='1' cores='12' threads='1'/>
</cpu>
```

Libvirt may normalize that topology by adding `dies='1'` and `clusters='1'`.
That is expected.

The guest also uses the AMD/Sequoia CPU model and feature mask derived from
OSX-PROXMOX. Keep that value in the per-domain QEMU command line and verify the
actual QEMU launch arguments after upgrades. Do **not** copy a CPU definition
from an unrelated Intel or Proxmox configuration.

## GPU passthrough

Pass through both functions of the RX 6600M. Do not retain a virtual display
adapter or SPICE graphics device in the same domain. A minimal template is in
[docs/gpu-passthrough.xml](docs/gpu-passthrough.xml).

After starting the guest, validate from the host:

```fish
sudo virsh -c qemu:///system domstate macOS
lspci -nnk -s 65:00.0 -s 65:00.1
sudo tail -80 /var/log/libvirt/qemu/macOS.log
```

In macOS, check **System Settings -> General -> About -> System Report ->
Graphics/Displays**. The RX 6600M should appear and Metal should be supported.

## Using the GPD Duo upper panel

The upper panel accepts USB-C DisplayPort Alt Mode video input. Connect the
**DisplayPort output on the eGPU** to that USB-C input with a bidirectional
DisplayPort-to-USB-C cable that explicitly supports DP-to-USB-C displays.

Do not connect the upper panel to the eGPU enclosure's host/upstream USB4 or
OCuLink port. Keep the GPD Duo powered because it powers the panel. The panel
does not provide audio over that input; use another audio route.

Until the cable is available, macOS Remote Management or Screen Sharing is a
useful headless fallback. Enable it before removing the virtual display.

## Networking and recovery-server troubleshooting

The libvirt `default` NAT network must be active and persistent:

```fish
sudo virsh -c qemu:///system net-list --all
sudo virsh -c qemu:///system net-info default
```

The network in this configuration is `virbr0` on `192.168.122.0/24`. On hosts
using UFW, allow DHCP and DNS traffic from `virbr0` to libvirt's dnsmasq and
allow routed traffic from `virbr0` to the host's internet-facing interface.
Otherwise macOS may obtain no lease, or reach IP addresses while failing DNS.

Diagnostics:

```fish
sudo virsh -c qemu:///system net-dhcp-leases default
sudo ss -ulnp | rg ':(53|67)\\b'
sudo ip -4 addr show virbr0
sudo timeout 30 tcpdump -ni virbr0 -vv '(udp port 67 or udp port 68 or udp port 53)'
```

If the installer reports that the recovery server cannot be contacted, first
verify a DHCP lease, a default route, direct IP connectivity, and DNS from the
guest. Do not change SMBIOS values repeatedly while the network path remains
unverified.

## USB input passthrough

Pass through whole external USB devices or their dedicated receivers, not the
GPD's built-in keyboard, trackpad, Bluetooth controller, camera, or fingerprint
reader. Keep a host input device available: passed-through devices disappear
from CachyOS while the VM is running.

Use stable vendor/product IDs in the persistent XML. Avoid pinning USB bus and
device numbers because they often change after reconnecting or rebooting.

## Power management

Prevent macOS from automatically sleeping while the VM has no physical display
attached. A guest can enter `pmsuspended`, which is distinct from a normal
libvirt pause. Try this first:

```fish
sudo virsh -c qemu:///system dompmwakeup macOS
```

If libvirt reports `pmsuspended` but QEMU rejects the wake request, the state
is inconsistent. A normal stop/start is the practical recovery; it discards
the suspended RAM state but does not delete the VM or its disks.

## Verification checklist

```fish
sudo virsh -c qemu:///system dominfo macOS
sudo virsh -c qemu:///system dumpxml macOS \
  | rg -n -C 1 '<memory|<currentMemory|<vcpu|<cpu|<topology|hostdev|video'
sudo virsh -c qemu:///system domiflist macOS
```

Confirm all of the following before considering the configuration complete:

- VM is `running` with 12 vCPU and 32 GiB memory.
- Both GPU functions are bound to `vfio-pci` on the host.
- No virtual video device is present.
- macOS reports Metal support for the RX 6600M.
- Guest networking has a DHCP lease and working DNS.
- Keyboard and mouse work in macOS while a separate host input method remains
  available.

## Related references

- [OSX-PROXMOX](https://github.com/luchina-gabriel/OSX-PROXMOX)
- [libvirt PCI host device assignment](https://libvirt.org/formatdomain.html#host-device-assignment)
- [libvirt virtual networks](https://libvirt.org/formatnetwork.html)
