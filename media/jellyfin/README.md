# Jellyfin setup

This folder contains both docker-compose files for Jellyfin.

# Navigation
- [Apps](/apps/)
- [Media Server](/media/)
  - [Jellyfin](/media/jellyfin/)
    - [The Differences](#the-differences)
    - [Proxmox Unprivileged LXC Hardware Acceleration](#proxmox-unprivileged-lxc-hardware-acceleration)
    - [iOS Apps](#ios-apps)
    - [Plugins](#plugins)
    - [Transcoding](#transcoding)
- [Remote Access](/access/)

# The Differences
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

# iOS Apps

This is just a short list of the ios apps i use for watching jellyfin on my phone or on the go.


### Primary App: Streamyfin

**Developer:** Streamyfin
**Source:** [GitHub](https://github.com/streamyfin/streamyfin)

Streamyfin is my go-to iOS Jellyfin client. It’s fast, actively developed and packed with features, including:
  
  •	Native integration with Jellyseerr for seamless media requests
  
  •	Support for the Skip Intro plugin
	
  •	Clean and intuitive mobile-first UI
	
  •	Frequent updates and improvements

This app strikes the perfect balance between usability and power, making it ideal for daily use.


### Backup App: Swiftfin

**Developer:** jellyfin
**Source:** [GitHub](https://github.com/jellyfin/Swiftfin)

While more minimal, it’s reliable and stable, essentially offering:
  
  •	A native iOS experience built on top of Jellyfin’s web interface
  
  •	Smooth playback and navigation
  
  •	Fewer dependencies, good fallback when Streamyfin has issues

I keep Swiftfin installed as a fallback option. It’s simple, clean, and gets the job done when needed.


### Why This Matters

Having multiple clients ensures I can enjoy my media with minimal interruption, no matter what. Streamyfin handles 95% of my usage, while Swiftfin is there to back me up.


# Plugins

This section contains all the plugins i use for my instance(s). Some are merely for a little improvement and some are for basic usage.

**AudioDB**

**MusicBrainz**

**OMDb**

**Open Subtitles**

**TMDb**

**Studio Images**

**TVmaze**

**TheTVDB**

### Custom Repositories

**Merge Versions**
[GitHub](https://github.com/danieladov/jellyfin-plugin-mergeversions)

**Intro Skipper**
[GitHub](https://github.com/intro-skipper/intro-skipper)

[Manifest](https://manifest.intro-skipper.org/manifest.json)


# Transcoding
My transcoding setup

**Intel QuickSync (QSV)**

Decoding for:

✅ H264

✅ VC1

✅ HEVC 10bit

✅ VP9 10bit

----

**Hardware Encoding:** Enabled

**Allow encoding in HEVC format:** Enabled

**Tone Mapping:** Disabled

**Transcoding thread count:** Auto

### Throttle Transcodes: Enabled
Throttle Transcodes is really useful as you will buffer alot less. Basically without it, Jellyfin will transcode the entire media file while watching which is not ideal but having it enabled will only transcode near where you're watching and not stressing itself trying to get ahead.

I recommend Throttling Transcodes, you can test if it's better or worse for you.