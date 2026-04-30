---
layout: post
title: "How I Hacked Back Into My Server Through a Media Player"
date: 2026-04-17
categories: homelab linux debugging
---

_A headless Arch server, a kernel update that broke ZFS, no physical access, and the only way in was through a media player._

---

## The Setup

I run a headless Arch Linux server at home. ZFS for bulk storage, a few containers, some VMs. It sits on my LAN, accessible only via SSH with pubkey auth through a WireGuard VPN. No monitor, no keyboard, no IPMI.

I was stuck abroad for weeks, possibly longer. No one who understands Linux could physically access the machine.

---

## The NIC Dies

Around mid-March, the server went dark. No SSH, no ping, nothing. Other devices on the LAN were reachable through my VPN, so the network was fine. The server had simply vanished.

It was the onboard Intel I217-LM - its transmit descriptor queue froze, a known bug with PCIe ASPM and the Intel ME arbiter. The PHY stayed up (link light on), so the kernel never triggered a reset. The NIC just sat there, link up, passing zero packets. This had happened once before, about a year earlier - same fix, unplug and replug the cable. I hadn't thought about it since.

I asked a friend to go to my apartment. I unlocked the door remotely using a SwitchBot smart lock - one of those IoT gadgets that's more useful than you'd expect. He unplugged and replugged the Ethernet cable, and plugged in a USB WiFi dongle as a backup path. The NIC recovered. I was back in.

Over the next few days I hardened things remotely: a watchdog script that detects TX stalls and resets the interface, WiFi failover, and `pcie_aspm=off` in GRUB. The system was stable. I ran `pacman -Syu`, verified everything, and rebooted to apply the kernel update and boot parameters cleanly.

---

## Locked Out

The server came back. It responded to pings. But SSH rejected every key I threw at it. `Permission denied (publickey)`.

I spent some time trying everything from the client side. Different keys, different algorithms, forcing specific signature types. I generated fresh keys and tried those. I tried from my laptop, from the OpenWrt router on the same LAN. I watched the verbose SSH output cycle through every key in my agent - seven of them - each one offered and rejected, until the server disconnected me with `Too many authentication failures`. I stripped it down to a single key. Still rejected. I forced algorithm negotiation. Still rejected.

I didn't know what was wrong. The SSH config could have changed during the update, though there was no obvious reason it would - I hadn't touched it, and `pacman` doesn't overwrite modified config files. Permissions on my `.ssh` directory could have shifted. The new OpenSSH version might have dropped support for my key type. I was debugging blind, with no way to see the server side of the conversation.

The only clue was that `sshd` was clearly running and accepting connections - it just wouldn't let me in.

I was weeks away from getting home, at minimum. I had personal documents on that server I needed, ongoing side projects, and a Home Assistant VM that controlled parts of my home - covers, windows, power, climate and more - which I relied on managing remotely. I had to get back in.

I have friends nearby, but walking someone through headless SSH debugging over the phone is a nightmare for everyone involved.

---

## Finding a Way In

I started checking what else was reachable on the server. Port by port, service by service. Most things were down - my containers hadn't survived the reboot, Samba was broken, Home Assistant was gone.

But one service was up: Jellyfin, my media server, listening on port 8096. I could reach the web UI through my VPN and log in. Jellyfin wasn't exposed to the internet - it was only reachable through my VPN.

A few weeks earlier, I'd been tinkering with Jellyfin's plugin system and its REST API for an unrelated project. I'd generated an API key, played with the endpoints, and gotten a sense of how plugins are loaded - .NET assemblies that Jellyfin discovers and runs inside its own process with the service user's permissions.

I decided to write a Jellyfin plugin that runs shell commands and reports back the output.

### Building a diagnostic plugin

I don't know C#. With an LLM doing the heavy lifting, I wrote a minimal Jellyfin plugin - an `IHostedService` that runs shell commands on startup and writes the output to Jellyfin's log directory, where I could read it through the API.

Getting the plugin onto the server was a mess. I couldn't build .NET locally, so I used GitHub Actions. Each iteration - push code, wait for the build, create a release, update the plugin manifest, install via the API, restart Jellyfin, read the output - took 5 to 8 minutes when everything went right. When something went wrong, add another 10.

But when the output finally came back, the problem was obvious:

```
ls: cannot access '/home/myuser/.ssh/': No such file or directory
```

```
The ZFS modules cannot be auto-loaded.
```

```
dkms status: zfs/2.3.3: added
find /lib/modules -name 'zfs.ko*': (empty)
```

My home directory didn't exist. Not corrupted, not permission-denied - _gone_.

The reason: ZFS's CDDL license is incompatible with the kernel's GPL, so it can never ship in mainline. On Arch, it's built out-of-tree via DKMS, which means every time the kernel updates, the ZFS module needs to be recompiled against the new kernel headers. I'd actually run into a ZFS mount issue once before after a kernel update - that time the module built fine but the mount service lost a race with the bind mounts. I'd fixed it with a systemd dependency and moved on.

This time was worse. The LTS kernel had jumped from 6.15 to 6.18, and ZFS 2.3.3 simply didn't support 6.18. DKMS tried to compile the module, failed, and told nobody. No alert, no failed boot warning, no email. The system booted cleanly into a state where a critical storage layer silently didn't exist.

No ZFS module meant no pool import. No pool meant no `/home`. No `/home` meant no `authorized_keys`. SSH kept looking for keys in a file that wasn't there.

### From diagnostics to a shell

Waiting 8 minutes to see each diagnostic output was slow going. So the next iteration replaced the diagnostic dump with an interactive HTTP endpoint - a controller that accepts a command via POST and returns stdout, stderr, and exit code. Once the plugin finally deployed: instant command execution. I could now run any command on the server by sending a curl request to the plugin endpoint and reading the response.

The first thing I checked was who the Jellyfin process was running as:

```
uid=973(jellyfin) gid=973(jellyfin) groups=973(jellyfin),91(video),1000(myuser)
```

The Jellyfin user was in my user's group - a workaround I'd set up ages ago so Jellyfin could access some family videos stored under my home directory. But I needed root to fix the storage layer. The interactive endpoint supported piping stdin, which meant I could feed a password to `su`. A few curl commands later, I had root access tunneled through the plugin's endpoint over my VPN.

### Getting SSH back

With root, I first tried to fix the real problem - rebuild ZFS for the new kernel. No luck. The third-party package repo was months behind. That would have to wait.

So I focused on the immediate goal: get SSH access back. My storage layout was the key. `/home` was a bind mount from `/data/home` on the root filesystem. Normally, the ZFS pool mounts over `/data` and provides the real content. But without ZFS, `/data/home` was an empty directory on XFS - empty, but writable.

I created a temporary `.ssh` directory via the plugin endpoint, copied my public key from my laptop through it, and set the right ownership and permissions:

```bash
mkdir -p /data/home/myuser/.ssh
# copied authorized_keys content from my laptop via the plugin endpoint
chmod 700 /data/home/myuser/.ssh
chmod 600 /data/home/myuser/.ssh/authorized_keys
chown -R 1000:1000 /data/home/myuser/.ssh
```

Start sshd. Try the connection.

```bash
$ ssh myuser@server
Welcome to Arch Linux!
```

I was in. From this point on, the rest was standard Linux troubleshooting.

---

## The Fix

With proper SSH access, everything became routine sysadmin work.

The third-party `archzfs` repo was months behind upstream, but the AUR had an OpenZFS package built for my exact kernel. After some dependency wrangling, I built and installed it.

```bash
$ sudo modprobe zfs
$ sudo zpool import
  pool: data
    state: ONLINE
```

The pool was healthy. Every byte intact. Months of data, configs, the real `authorized_keys`, all safe.

Before rebooting, I set up a proper safety net this time: temporary password auth and root login as a fallback, rebuilt initramfs with the ZFS hook, verified the boot chain end to end. I wasn't going to get locked out twice.

```bash
$ sudo reboot
```

Thirty seconds later:

```bash
$ ssh myuser@server
```

In. ZFS mounted. Home directory intact. Everything back.

---

## What I Learned

**Always have a second way in.** I need a PiKVM, or an IPMI card, or a serial console - out-of-band access that doesn't depend on the OS being healthy. Everything else - the silent DKMS failure, the ZFS license situation, the storage-dependent SSH keys - is a minor inconvenience if you can get to a console. Without one, I was stuck building C# plugins for a media server at 2am with an AI assistant, piping shell commands over curl through a movie streaming API.
