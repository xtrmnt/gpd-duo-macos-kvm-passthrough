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
| eGPU | SGWZONE RX 6600 eGPU, connected to the GPD Duo over USB4 |
| GPU | AMD Navi 23 / Radeon RX 6600M, plus its HDMI/DP-audio function |
| Network | libvirt `default` NAT network, VMXNET3 |
| Input | Direct USB passthrough for a wired Razer mouse and Dell keyboard |

## Physical desktop layout

The GPD Duo remains the CachyOS host. Its lower display is the CachyOS screen.
The SGWZONE RX 6600 eGPU is connected through the GPD Duo's USB4 port, and its
Radeon RX 6600M is passed through to the macOS VM. In the completed physical
layout, the GPD Duo's upper display is the macOS screen: it receives the
eGPU's DisplayPort output through a bidirectional DisplayPort-to-USB-C cable
and its USB-C DisplayPort Alt Mode input.

```text
GPD Duo lower display  -> CachyOS host
GPD Duo USB4 port      -> SGWZONE RX 6600 eGPU
SGWZONE DisplayPort    -> GPD Duo upper-display USB-C video input
GPD Duo upper display  -> macOS VM (RX 6600M passthrough)
```

This layout keeps the host iGPU available to CachyOS and reserves the external
RX 6600M exclusively for macOS.

## Scope and important limits

- Check Apple's current macOS license terms before using macOS in a virtual
  machine or on non-Apple hardware.
- You are responsible for your own hardware, data, firmware, and configuration
  choices. The repository author accepts no responsibility for damage, data
  loss, downtime, or other consequences, and does not provide individual
  troubleshooting or support.
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

## Prerequisites checklist

Validate these before creating or changing the VM. A green check against this
list avoids most opaque boot, networking, and passthrough failures.

### Firmware and hardware

- [ ] Update the GPD Duo BIOS to the latest stable version supplied by GPD
  before beginning. Complete the firmware update on reliable power and retain
  the vendor's recovery guidance.
- [ ] AMD SVM/virtualization and IOMMU are enabled in the GPD Duo firmware.
- [ ] **Resizable BAR is disabled.** This was required by the working
  configuration documented here.
- [ ] The host Radeon 890M remains available to CachyOS for its own display.
- [ ] The eGPU is attached, powered, and visible as a separate PCI device
  before the VM is started.
- [ ] The eGPU graphics and audio functions are in usable IOMMU groups; neither
  group contains a device that the host needs.
- [ ] A recovery path is available: a host display/input device, SSH session,
  or a second computer. Do not pass through the only host keyboard and mouse.

### CachyOS host

- [ ] KVM/QEMU, libvirt, OVMF/edk2 firmware, dnsmasq, and nftables support are
  installed and functional.
- [ ] The system libvirt connection works:

  ```fish
  sudo virsh -c qemu:///system list --all
  ```

- [ ] The OVMF files exist:

  ```fish
  test -f /usr/share/edk2/x64/OVMF_CODE.4m.fd
  test -f /usr/share/edk2/x64/OVMF_VARS.4m.fd
  ```

- [ ] A writable per-VM VARS file is used. The system `OVMF_VARS.4m.fd` file is
  a template, not VM state.
- [ ] VFIO is included in the initramfs and the passthrough device IDs are
  bound to `vfio-pci`; see [CachyOS boot and VFIO configuration](#cachyos-boot-and-vfio-configuration).
- [ ] Host sleep is disabled or inhibited while the VM is running. An
  unattended host suspend can leave a macOS guest in a confusing power state.

### Storage safety

- [ ] The initial installation uses a disposable qcow2 system disk, not a
  physical host disk.
- [ ] The host boot disk, personal files, and sample-library partitions remain
  unattached until the base VM is stable and backed up.
- [ ] The recovery image and OpenCore image are stored outside any physical
  device planned for later passthrough.
- [ ] A backup of the libvirt domain XML and the qcow2 image exists before
  changing storage or PCI assignments.

### Networking and remote access

- [ ] The libvirt `default` NAT network exists, is active, and autostarts.
- [ ] If UFW is enabled, DHCP, DNS, and forwarding from `virbr0` are permitted
  before trying macOS Recovery.
- [ ] The guest receives a DHCP address on `192.168.122.0/24` and resolves DNS.
- [ ] Remote Management/Screen Sharing is enabled in macOS before virtual
  graphics are removed, or a physical display is ready.

### Display and USB

- [ ] For the GPD Duo's upper panel, use the eGPU's DisplayPort output and a
  bidirectional DP-to-USB-C cable that supports DP-to-USB-C display input.
- [ ] Do not use the eGPU enclosure's host/upstream USB4 or OCuLink connector
  as the panel's video source.
- [ ] Pass through only dedicated external USB devices or receivers; built-in
  camera, Bluetooth, fingerprint, keyboard, and trackpad should stay with the
  host.

## Host prerequisites

1. Enable AMD IOMMU in firmware and the host kernel.
2. Disable **Resizable BAR** (also shown as Re-Size BAR or Smart Access
   Memory) in the GPD Duo firmware before attempting the GPU passthrough boot.
   It was disabled in the working configuration documented here.
3. Confirm that the eGPU graphics and audio functions are isolated from the
   host, then bind both to `vfio-pci`.
4. Keep the GPD Duo's integrated Radeon 890M bound to the host `amdgpu`
   driver. It is not the passthrough GPU.
5. Confirm that the VM is shut off before adding the PCI devices.

Resizable BAR and **Above 4G Decoding** are separate options. This guide only
records the tested requirement to disable Resizable BAR; do not change Above
4G Decoding solely on the basis of this note.

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

### CachyOS boot and VFIO configuration

This working GPD Duo host uses a systemd-boot-style kernel command line in
`/etc/kernel/cmdline`; it does not use `/etc/default/grub`. Its command line
contains the normal CachyOS root and presentation options, with **no explicit**
`amd_iommu=on`, `iommu=pt`, or `vfio-pci.ids=` arguments. IOMMU is enabled by
firmware on this host, while the GPU binding is established before the normal
graphics driver loads.

Load the VFIO modules early through mkinitcpio:

```text
# /etc/mkinitcpio.conf
MODULES=(vfio_pci vfio vfio_iommu_type1)
```

Bind both functions of the passthrough GPU by PCI vendor/device ID:

```text
# /etc/modprobe.d/vfio.conf
# Tested RX 6600M graphics + HDMI/DP audio functions
options vfio-pci ids=1002:73ff,1002:ab28
```

Those IDs are specific to the tested RX 6600M. Replace both IDs with the
graphics and audio IDs reported by `lspci -nn` for any other GPU.

After changing either file, rebuild every installed initramfs and reboot:

```fish
sudo mkinitcpio -P
sudo reboot
```

After the reboot, confirm the intended state before starting the VM:

```fish
lsmod | rg '^vfio|^amdgpu'
lspci -nnk -s 65:00.0 -s 65:00.1
```

The RX 6600M functions must show `Kernel driver in use: vfio-pci`; the GPD
Duo's Radeon 890M must remain on `amdgpu`. Do not bind the 890M to VFIO.

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
libvirt pause.

### Wake a sleeping guest from CachyOS

Try the libvirt guest-wake request first:

```fish
sudo virsh -c qemu:///system dompmwakeup macOS
sudo virsh -c qemu:///system domstate macOS
```

If libvirt reports `pmsuspended` but QEMU rejects the wake request, the state
is inconsistent. The practical recovery is to discard the suspended RAM state
and start a clean VM process:

```fish
sudo virsh -c qemu:///system destroy macOS
sudo virsh -c qemu:///system start macOS
```

`destroy` does not delete the VM or its disks, but it is a forced stop and
discards guest RAM. Use it only after the normal wake request fails.

### Keep macOS awake

In macOS Terminal, apply these settings once. They disable system sleep,
display sleep, disk sleep, and automatic power-off for every macOS power
profile:

```bash
sudo pmset -a sleep 0
sudo pmset -a displaysleep 0
sudo pmset -a disksleep 0
sudo pmset -a autopoweroff 0
sudo pmset -g custom
```

### Keep the CachyOS host awake

Prevent the host from sleeping or responding to a lid-close event while the VM
is running:

```fish
sudo systemd-run \
  --unit=macos-vm-inhibit \
  --property=Type=simple \
  systemd-inhibit \
  --what=sleep:handle-lid-switch \
  --mode=block \
  --why="macOS VM is running" \
  sleep infinity
```

The inhibitor lasts until reboot or until it is explicitly removed:

```fish
sudo systemctl stop macos-vm-inhibit.service
```

### Desktop-shutdown helper

In normal operation, choosing **Apple menu -> Shut Down** emits a libvirt
lifecycle event (`Shutdown Finished after guest request`), and the domain
changes to `shut off`. Confirm it from CachyOS after a short wait:

```fish
sudo virsh -c qemu:///system domstate macOS
sudo tail -15 /var/log/libvirt/qemu/macOS.log
```

The expected results are `shut off` and a log entry similar to `shutting down,
reason=shutdown`.

The optional [scripts/macos-finish-shutdown](scripts/macos-finish-shutdown)
helper is an emergency fallback only. Use it if the macOS shutdown request has
already been made, a reasonable wait has elapsed, and libvirt still reports the
VM as `running`. It waits for a configurable grace period, then calls `virsh
destroy` only if the VM still reports `running`. It never deletes the VM or its
disks, but `destroy` discards guest RAM, so it must not be part of the routine
shutdown path.

Install it on the host:

```fish
sudo install -Dm755 scripts/macos-finish-shutdown /usr/local/sbin/macos-finish-shutdown
```

Only for the stuck-shutdown case described above, run:

```fish
sudo macos-finish-shutdown
```

The default grace period is 90 seconds. Pass a different number of seconds if
needed, for example `sudo macos-finish-shutdown 120`.

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

## Planned enhancement: seamless input with Deskflow

The current input arrangement uses direct USB passthrough for the external
keyboard and mouse. A future improvement is to use
[Deskflow](https://github.com/deskflow/deskflow) to share the GPD Duo's
built-in keyboard and trackpad from CachyOS to macOS over the VM network.

The intended experience is to move the pointer from the lower CachyOS display
to the upper macOS display and continue typing without switching USB devices.
Keep this as an optional post-install enhancement: it should be configured
only after GPU output, networking, and Remote Management are stable. The
built-in GPD input devices remain owned by CachyOS; Deskflow shares input at
the desktop level rather than passing those devices through to the VM.

## Related references

- [OSX-PROXMOX](https://github.com/luchina-gabriel/OSX-PROXMOX)
- [libvirt PCI host device assignment](https://libvirt.org/formatdomain.html#host-device-assignment)
- [libvirt virtual networks](https://libvirt.org/formatnetwork.html)
