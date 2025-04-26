# Jellyfin setup

This folder contains both docker-compose files for Jellyfin.


### The Differences
Jellyfin supports transcoding just like plex. So instead of fully converting the file using something like tdarr (also in this repo) it transcodes as you watch to a different quality or codec. This is very taxxing on hardware just like an Intel CPU (QSV/Quicksync) or NVIDIA GPUs (NVENC/CUDA).

**hwaccel-compose.yaml** - This is the Hardware acceleration version. This is designed for use with Proxmox.

**noaccel-compose.yaml** - This has no hardware acceleration mounts and is just the normal Jellyfin compose file. It can be later modified to suit your own needs.


# Proxmox Unprivileged LXC Hardware Acceleration

Adding your existing NVIDIA GPU or Intel QSV (or other) from Proxmox into your LXC container, you can use the following commands to set this up.

**Step 1: Find the Device**

Run on Proxmox host:
```
lspci | grep -i vga
```
Look for something like:
```
00:02.0 VGA compatible controller: Intel Corporation HD Graphics 620 (rev 02)
```
Check for the iGPU render node:
```
ls -l /dev/dri
```
Expected output:
```
/dev/dri/card0
/dev/dri/renderD128
```
**Step 2: Edit the Container Config**

Edit your container config (replace 100 with your container ID):
```
nano /etc/pve/lxc/100.conf
```
Add the following lines:
```
# Intel iGPU passthrough
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```
**Step 3: Restart Container**
```
pct restart <container-id>
```
**Step 4: Verify Inside Container**

Inside your LXC container, run:
```
ls /dev/dri
```
You should see:
```
card0
renderD128
```

You may need to install VAAPI drivers:
```
apt install vainfo intel-media-va-driver-non-free
```
