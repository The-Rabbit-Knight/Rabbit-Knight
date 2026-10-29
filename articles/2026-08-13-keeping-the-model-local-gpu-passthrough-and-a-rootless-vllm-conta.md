---
title: "Keeping the Model Local: GPU Passthrough and a Rootless vLLM Container on Proxmox"
date: 2026-08-13
medium_url: https://medium.com/@rabbit_knight/keeping-the-model-local-gpu-passthrough-and-a-rootless-vllm-container-on-proxmox-118a2136f82f
---

> **Original Published Article on Medium:** [Keeping the Model Local: GPU Passthrough and a Rootless vLLM Container on Proxmox](https://medium.com/@rabbit_knight/keeping-the-model-local-gpu-passthrough-and-a-rootless-vllm-container-on-proxmox-118a2136f82f)

### Keeping the Model Local: GPU Passthrough and a Rootless vLLM Container on Proxmox

![Article Figure](https://cdn-images-1.medium.com/max/800/1*YJNdnTTp416uYB8D0fJGSA.png)

Anyone who reads my work knows I prefer hands-on experience, so with the sheer momentum behind AI right now, diving into local models was the natural next step. The goal of this piece is to build a fully functional local API endpoint to serve as our gateway to local LLMs.

I had the right hardware sitting idle in my lab: an AMD Radeon RX 7900 XTX with 24GB of VRAM, more than capable of running a genuinely useful model at a workable quantization. The challenge was logistics. I needed a clean way to pass that GPU through to a workload without destabilizing my existing Proxmox host or the VMs currently relying on it.

This article outlines the exact path I took to get from a GPU sitting idle to a locally reachable, authenticated, encrypted model endpoint, and the reasoning behind each choice along the way.

One thing before any of this starts. I’m not writing this as the correct or only way to do it. It’s the path that worked given my hardware, my existing infrastructure, and one hard constraint: I couldn’t afford to risk the VMs already running on this host. Where there was a real alternative, I’ve said what it was and why I didn’t take it. Not because my way is right. Because the reasoning is more useful to you than a recipe would be.

#### Plan at a glance

```
Proxmox VE 8.x.x host (existing VMs untouched)
  └─ VM 101 — RX 7900 XTX passed through via VFIO (03:00.0 + 03:00.1)
        └─ Ubuntu Server 24.04.4 LTS, HWE kernel
              └─ User added to the render group (rootless GPU access)
                    └─ Podman — rootless, daemonless container runtime
                          └─ vLLM ROCm container
                                ├─ /dev/kfd + /dev/dri passed 
in
 (--device ...:rwm)
                                ├─ TLS: self-signed cert, key locked to owner only
                                ├─ API key required on every request
                                └─ Qwen2.5-14B-Instruct-AWQ, GPU-accelerated
                                      └─ verified: https://localhost:8443/v1/chat/completions
```

#### Before any of this

A few things need to already be true on the Proxmox host before any of the VM setup below will work. I confirmed all of these on mine first. A missing one here is the difference between a clean setup and a few hours of troubleshooting something further down the chain that has nothing to do with what’s actually wrong.

IOMMU needs to be on, both in the BIOS and on the kernel command line.

```
cat
 /proc/cmdline
```

Look for iommu=pt on AMD, or intel_iommu=on if the host is Intel instead. Without this, the hardware level device isolation the whole VM approach depends on isn’t available at all.

The VFIO kernel modules need to be loading at boot.

```
cat
 /etc/modules
lsmod | grep vfio
```

vfio, vfio_iommu_type1, vfio_pci, and vfio_virqfd should show up in both.

The GPU’s normal driver needs to be blacklisted at the host level, so the host never claims the card before the VM gets a chance to.

```
cat
 /etc/modprobe.d/blacklist.conf
```

amdgpu should be sitting in there. If the host has already grabbed the card, lspci -nnk against its PCI address will show a driver already bound to it, and passthrough won’t behave until that’s cleared up first.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*IVoDuyYcdOavOY8OOfOfow.png)

Worth checking the IOMMU groups too, before committing to this path at all.

```
for
 d 
in
 /sys/kernel/iommu_groups/*/devices/*; 
do
  n=
${d#*/iommu_groups/}
; n=
${n%%/*}
  
printf
 
"Group %s: "
 
"
$n
"
; lspci -nns 
"
${d##*/}
"
done
```

Ideally the GPU, and its HDMI or DisplayPort audio function, sit alone in their own group, nothing else sharing it. If something unrelated is in there with it, that’s solvable with an ACS override patch, but it’s added complexity worth avoiding if there’s a choice.

Last one, just storage. Whatever Proxmox pool ends up holding the new VM’s disk needs the room for it.

```
pvesm status
```

One thing I’m leaving off this list on purpose. An isolated network segment isn’t a prerequisite for anything in this piece. That only matters once this endpoint needs to be reached from somewhere else, and that question isn’t part of this story.

#### The lab

```
Component         | Detail
------------------|----------------------------------------------------------
Hypervisor        | Proxmox VE 
8.
x.x
GPU               | AMD Radeon RX 
7900
 XTX (Sapphire Nitro+ Vapor-X), 
24
GB VRAM
VM                | Ubuntu Server 
24.04
.4
 LTS, HWE kernel
VM resources      | 
8
 cores (host 
type
), 
48
GB RAM, 
128
GB disk
Container runtime | Podman, rootless
Inference server  | vLLM, ROCm build
Model             | Qwen2
.5
-14
B-Instruct, AWQ, 
4
-bit
```

#### The isolation decision

Proxmox gives you two ways to hand a GPU to a workload: an LXC container (which shares the host’s kernel) or a VM with the GPU passed through at the hardware level (via VFIO). LXC is lighter-weight and easier to set up, but that shared kernel is exactly the problem: if anything running inside the container is compromised, the attacker is one kernel exploit away from the Proxmox host itself, and everything else running on it. A VM has its own kernel. A compromise there requires an actual hypervisor escape to reach the host, which is a fundamentally harder and rarer class of vulnerability. So I went with a VM with PCIe passthrough, not an LXC container.

#### Building the VM

A few settings here aren’t defaults, and each exists for a reason.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*dlwIA0u_Bit7c4NmpkDr-A.png)

Machine type q35, not the older i440fx. q35 models PCIe properly, and passthrough on the older chipset emulation is unreliable enough that it isn’t worth fighting. BIOS set to OVMF, Proxmox’s UEFI firmware, which passthrough needs far more reliably than legacy SeaBIOS. SCSI controller set to VirtIO SCSI single instead of the default LSI emulation: better performance, more control per disk. I added a virtual TPM too, on the same storage as the EFI disk. Ubuntu doesn’t strictly need it. It was cheap to include and kept the door open in case anything later in the stack expected one.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*zl1R09ziehoSj1AQeahOsQ.png)

Disk cache mode is write-back, and Proxmox marks it “unsafe” right there in the UI. That label is accurate, and the trade is deliberate. Write-back gives up some crash consistency for real gains in disk I/O, a fair trade when nothing on the disk is irreplaceable (remember this is an experiment). The OS and the containers on it are trivial to rebuild if something goes wrong. I would not make the same trade on a VM holding data I couldn’t regenerate.

CPU type is host, not an emulated generic model, so the guest gets the full instruction set of the physical chip. Matters for performance, and for some inference code paths that lean on newer extensions. Memory ballooning is off. A GPU compute VM is exactly the workload where you don’t want the hypervisor quietly reclaiming memory mid-run.

#### Ubuntu, and the kernel that actually matters

![Article Figure](https://cdn-images-1.medium.com/max/800/1*N1SoOVVDFi_F20KxqLELJA.png)

Standard Ubuntu Server 24.04 install, with one deliberate choice along the way: the HWE kernel instead of the GA kernel that ships by default. HWE tracks newer upstream kernel releases inside the same LTS branch, and AMD’s driver support for recent cards gets meaningfully better in those newer versions. This is a current-generation discrete card. The GA kernel probably still works, but the HWE kernel hands the driver more recent code to work with, and passthrough is not somewhere I wanted to be debugging against an older driver if I didn’t have to be. SSH went in during install too. This box has been headless since the first boot.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*GvNub-85cBAcaeavfAeDzA.png)

#### Passing the card through

Before Proxmox can hand the GPU over, it needs to know exactly which device to hand over.

```
lspci | grep VGA
```

That put the 7900 XTX at PCI address 03:00.0 (the motherboard assigns the graphics card a specific hardware address) on the host. Hold onto that number. It’s about to stop being true the moment you look at it from inside the guest.

With the address in hand, I edited the VM’s config file directly instead of using the PCI device dialog in the Proxmox GUI.

```
nano /etc/pve/qemu-server/101.conf
```

The GUI handles straightforward passthrough fine, but one setting that mattered here, romfile, isn't cleanly exposed there. It hands the VM a direct copy of the card's own vBIOS, the small piece of firmware a GPU runs through the instant it powers on, before any operating system is even involved. Some GPUs initialize more reliably inside a VM when given their own vBIOS dump instead of relying on the VM's default ROM handling. I'd solved this exact problem once before on this same host, for a different card, so I checked whether a dump already existed before making a new one.

```
find /var -name 
'vbios_7xxx.bin'
```

Found one, at /usr/share/kvm/vbios_7xxx.bin, the directory Proxmox looks in for ROM files. That dump belonged to a different card, though, and each passed-through GPU should carry a dump of its own vBIOS, not a borrowed one. So the 7900 XTX’s own dump went into that same folder under its own name, vbios_XTX.bin, so it wouldn’t get mixed up with the other card’s file sitting right next to it.

```
# add to congig nano /etc/pve/qemu-server/101.conf
hostpci0: 0000:03:00.0,pcie=1,romfile=vbios_XTX.bin
hostpci1: 0000:03:00.1,pcie=1
```

Two lines, not one. A card like this exposes two separate PCI functions: 03:00.0 is the compute and display engine, 03:00.1 is its HDMI and DisplayPort audio device. Both need to go to the guest together or the card doesn’t work right.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*tbhUx2DCAK-RxsnYtGFxxw.png)

No x-vga=1 in there. That flag makes the passed-through card the VM’s primary display, the right call for a gaming VM and the wrong call here. This workload never displays anything. It computes. I left Proxmox’s default console in place for management access, and the 7900 XTX exists purely as a second device for the guest’s own driver to claim once it’s installed.

#### An address that isn’t the address

A quick note on what a PCI address like 03:00.0 actually is. Every piece of hardware plugged into a motherboard, the GPU included, sits at a specific slot on an internal map the system builds when it boots. Bus 03, device 00, function 0 is just this particular machine's way of saying "the graphics card lives here." It isn't a name the card carries around with it. It's a coordinate, assigned fresh by whatever system happens to be looking at it.

That distinction shows up directly in the next check, run to confirm the container would actually be able to reach the GPU.

```
ls
 -la /dev/kfd
find /dev/dri
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*-CulF__1iOhl9jk44k066A.png)

Look at those by-path lines. pci-0000:01:00.0, not 03:00.0. Same physical card that got passed through a few minutes earlier using the 03:00.0 address, now showing up under a completely different number.

It is expected. The guest builds its own internal hardware map from scratch the instant it boots, with no knowledge of the host’s map and no reason to reuse its numbering. 03:00.0 is where Proxmox, standing on the host side, goes looking for the card to hand it over in the first place. Once that handoff happens, the guest is free to slot the card in wherever its own internal bus layout puts it, and in this case that turned out to be 01:00.0. The hostpci0 line in the VM’s config will always use the host’s number. Anything checked from inside the guest afterward, this find /dev/dri output included, uses the guest’s own number instead.

#### Choosing Podman

For the container runtime, Podman over Docker, and the reason is architectural. Docker runs a persistent background daemon with root privileges. That daemon is part of your attack surface for as long as the host is up, whether or not anything is actively running. Podman doesn’t run a daemon at all. Containers are just child processes of whatever started them, and it supports running the whole stack, GPU access included, under a standard, unprivileged user account.

#### The permission wall

Both /dev/kfd and /dev/dri existing at all, in the check above, was what confirmed the AMDGPU driver inside the guest had already claimed the card. Both were owned by root:render, though. Podman needs explicit permission flags on GPU devices (device /dev/kfd:rwm, granting read, write, and mknod), and rootless Podman running as an ordinary user doesn’t automatically get access to a device it isn’t in the group for. That’s Linux’s permission model doing exactly what it’s supposed to (a rootless container inheriting it is correct behavior). The fix was adding the user to the render group.

```
sudo nano /etc/group
# check
groups
```

#### Locking the door first

Before starting the real container, I set up TLS. Not because anything demanded it at this stage, but because there was no good reason for the inference API to speak plain HTTP even for local testing. Encrypting the connection is cheap. Doing it now means it’s never something bolted on later under pressure.

```
mkdir
 -p ~/vllm-certs
openssl req -x509 \
  -newkey rsa:4096 \
  -keyout key.pem \
  -out cert.pem \
  -sha256 \
  -days 365 \
  -nodes \
  -subj 
"/CN=localhost"
chmod
 600 key.pem
```

- newkey rsa:4096 generates a new 4096-bit key alongside the certificate, strong enough for something valid a full year (365 days is a concession to convenience, not strict security best practice).
- nodes leaves the private key unencrypted, no passphrase. The alternative means vLLM hangs on startup waiting for a password typed into a service with no interactive terminal attached, which doesn’t work for something meant to run in the background indefinitely.
- subj “/CN=localhost” sets the certificate’s identity to localhost, correct for testing from the same machine, and something that will need to change to the machine’s real address the moment this gets reached from anywhere else.
- chmod 600 restricts the private key to the owning user alone. Anyone who can read that file can impersonate the server, which makes it the one file in this whole setup that should never be group or world readable.

The certs directory lives under the home folder, not a system path like /etc/ssl/certs, keeping things consistent with running the stack rootless. A system-owned directory would just reintroduce the same permission friction the render group fix was solving a minute earlier.

#### Running the container

```
podman run -d 
--name vllm \
  
--device /dev/kfd:rwm --device /dev/dri:rwm \
  
--group-add keep-groups \
  
--security-opt seccomp=unconfined \
  
--ipc=host \
  -p 
8443
:
8000
 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -v ~/vllm-certs:/certs:ro \
  vllm/vllm-openai-rocm:latest \
  
--model Qwen/Qwen2.5-14B-Instruct-AWQ \
  
--quantization awq \
  
--max-model-len 8192 \
  
--gpu-memory-utilization 0.85 \
  
--api-key $(openssl rand -hex 32) \
  
--ssl-keyfile /certs/key.pem \
  
--ssl-certfile /certs/cert.pem
```

- device flags and group-add keep-groups are the permission fix from a minute ago, carried into the real run.
- security-opt seccomp=unconfined turns off the container’s default syscall filtering, and I want to be straight about what that means instead of skipping past it: current ROCm workloads commonly need syscalls the default seccomp profile blocks, so this widens what the container is allowed to do at the kernel level. The alternative was a container that couldn’t talk to the GPU at all, so I made the trade. It is a trade, though, not a free improvement.
- ipc=host shares the host’s shared memory namespace with the container. PyTorch and ROCm workloads often need this or they fail outright with out-of-memory errors on shared memory specifically. There’s a bounded alternative, shm-size, that keeps the container’s shared memory isolated but capped at a fixed amount. ipc=host just removes the boundary instead of sizing around it. Simpler, at the cost of one more piece of isolation given up.
- p 8443:8000 maps host port 8443 to vLLM’s default internal port of 8000. 8443 on the outside is just a nod to HTTPS, nothing mandatory about the number itself. The two volume mounts matter for different reasons.
- ~/.cache/huggingface means the roughly 10GB of model weights only ever download once, instead of every time the container gets recreated.
- The certs directory is mounted read only, so the container can read what it needs to serve HTTPS but can never touch the certificate or key even if something inside it gets compromised.
- 4-bit quantization is shorthand for shrinking a model’s internal numbers so they take up less memory, trading a bit of precision for a model that actually fits on hardware you own instead of a rack of enterprise cards. AWQ is one specific method of doing that shrinking, and it matters here specifically because vLLM only reads models packaged this way, or with a sibling method called GPTQ. Not GGUF, the format most people run into first through Ollama or LM Studio.
- quantization awq tells vLLM which method to expect.
- max-model-len 8192 caps how much conversation context vLLM allocates memory for, instead of defaulting to whatever the model’s absolute maximum supports. A smaller, deliberate number keeps memory usage predictable.
- gpu-memory-utilization 0.85 caps vLLM at 85% of the card’s VRAM, leaving headroom instead of letting it grab everything available.
- api-key, generated inline here with $(openssl rand -hex 32), requires a bearer token on every request. Without it, anyone who could reach the port at all could use the model with no check whatsoever. I’ll be honest about a downside of doing it inline like this: it only ever exists in shell history and in whatever podman inspect reports back afterward, and a fresh key gets generated every single time the container happens to get recreated, silently breaking anything already configured to use the old one. Generating it once, saving it to a locked-down file, and pointing at that file on every start would hold up better.

#### Proof of life

```
podman logs -f vllm
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*4Smw7HrUx5389Ba0_ptcbg.png)

Turned up a long list of registered routes: /v1/chat/completions, /v1/models, /health, a few dozen more, followed by the server reporting it had started. That route list is the FastAPI app underneath announcing every endpoint it serves. Seeing it means the model loaded and the server is actually listening, not just that the container happens to be running.

Then the real test.

```
curl -X POST https://localhost:8443/v1/chat/completions \
  -k \
  -H 
"Content-Type: application/json"
 \
  -H 
"Authorization: Bearer 
$(cat ~/vllm-key)
"
 \
  -d 
'{
    "model": "Qwen/Qwen2.5-14B-Instruct-AWQ",
    "messages": [
      {"role": "system", "content": "You are a highly skilled AI assistant."},
      {"role": "user", "content": "Explain the difference between a CPU thread and a process in one sentence."}
    ],
    "max_tokens": 150,
    "temperature": 0.7
  }'
 | jq .
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*i0jWx1-tKDTWJfdnfzo2Gw.png)

-k tells curl not to verify the certificate, expected here since a self-signed cert was never going to validate against a public certificate authority in the first place. That’s fine, not a problem to solve. A full, coherent answer came back in about three seconds, connection overhead included. For a model this size, that timing alone is a decent signal the GPU did the work and not the CPU. A 14 billion parameter model running on CPU alone would not answer that fast.

That was the goal for this stage. A model running entirely on hardware I control, reachable from the same machine, requiring a key to talk to it, and never once touching the internet to answer a question. What happens with that endpoint from here is its own story.
