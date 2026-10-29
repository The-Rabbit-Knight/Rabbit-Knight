---
title: "How to Enable Audio in Proxmox Virtual Machines Using SPICE and Virt-Viewer (Linux)"
date: 2025-06-15
medium_url: https://medium.com/@rabbit_knight/how-to-enable-audio-in-proxmox-virtual-machines-using-spice-and-virt-viewer-linux-c4b5a499f780
---

> **Original Published Article on Medium:** [How to Enable Audio in Proxmox Virtual Machines Using SPICE and Virt-Viewer (Linux)](https://medium.com/@rabbit_knight/how-to-enable-audio-in-proxmox-virtual-machines-using-spice-and-virt-viewer-linux-c4b5a499f780)

### How to Enable Audio in Proxmox Virtual Machines Using SPICE and Virt-Viewer (Linux)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*xNlzIypYK1y5N2jY3GRHsg.png)

If you’re running virtual machines (VMs) on Proxmox and want them to play audio, you’re not alone. Proxmox Web VNC viewer don’t pass audio to your browser, but there’s a simple way to get it working using SPICE and virt-viewer.

I’ll walk you through the process step by step, and if you follow along, you’ll soon be streaming audio from your VM just like you would on a regular computer.

What You’ll Need:

1. Proxmox VE setup with a running VM (in my case Linux)
2. SPICE client (virt-viewer) installed on your local machine (again Linux in my case)

#### Step 1: Configure the VM in Proxmox for SPICE and Audio

1. In the Proxmox web GUI, go to your VM and select the Hardware tab.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*lb3xwRy-92ds44UCgeYs6A.png)

2. Add an Audio Device :

![Article Figure](https://cdn-images-1.medium.com/max/800/1*LcmvmVlGzHjmsn-BopYaBQ.png)

3. Edit existing Display device if it’s set to Default or VGA to SPICE.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*n1LUtalsDeUyd27HdjF4rw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*9he4yfo9WbY5DSSPXlO1Tg.png)

Why SPICE? SPICE allows better interaction with the VM including audio, dynamic resolution, and clipboard sharing. Proxmox supports SPICE natively.

#### Step 2: Install virt-viewer on Your Local Machine

SPICE is a Protocol, not an Application: SPICE (Simple Protocol for Independent Computing Environments) is a communication protocol, similar to how HTTP is a protocol for web Browse. Just as you need a web browser (like Chrome or Firefox) to interpret and display web pages sent via HTTP, you need a SPICE client application to interpret and display the graphical, audio, and input data sent from your VM via the SPICE protocol. virt-viewer is that dedicated SPICE client.

SPICE operates on a client-server model.

- Server (Proxmox/VM): Proxmox sets up a SPICE server for your VM.
- Client (Your Machine): virt-viewer acts as the SPICE client, connecting to the SPICE server on your Proxmox host and receiving the streamed desktop, audio, and input data.

Since I’m using Ubuntu Linux, I can install the viewer directly from the standard repository using:

```
sudo apt install virt-viewer
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_cwStf78ceBvgK1Yppp47A.png)

#### Step 3: Boot Your Windows VM

Make sure your Windows virtual machine is powered on and you are logged into the desktop. You’ll need to be able to access a web browser inside the VM.

#### Step 4: Download the SPICE Guest Tools Installer

Installing SPICE Guest Tools on a Windows VM is quite straightforward, as it’s designed to be a standard Windows installer.

1. Open a web browser inside your Windows VM.
2. Navigate to the official SPICE project download page: https://www.spice-space.org/

![Article Figure](https://cdn-images-1.medium.com/max/800/1*LlC43dZ19Mmx1y-lRJvmRg.png)

3. On this page, look for the section related to “Windows binaries” or “Windows SPICE Guest Tools.”

4. Click this link to download the installer file.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*qaa7BB4D8VUq5C6mdlys7w.png)

5. Restart your VM

#### Step 5: Connect Using SPICE Console

1. In Proxmox, go to your running VM.
2. Click Console → SPICE .

![Article Figure](https://cdn-images-1.medium.com/max/800/1*hVCaHmz0kLgGN9k6MbctVQ.png)

3. A .vv file will be downloaded—this is a shortcut to your VM.

4. Open that file with Remote Viewer ( virt-viewer ).

You should now see your VM desktop, and if everything is configured correctly, audio will come through your local speakers.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*OdsvFWMW6mPNGUVBOUN7Gg.png)

#### Step 6: Test the Audio

Now that you’re connected:

- Try playing a test sound or a video.
- Make sure your local machine isn’t muted.
- You should hear audio from the VM.

#### Troubleshooting Tips

- No sound? Make sure the SPICE tools or guest additions are installed.
- Still no audio? Check that the right audio device is selected in the VM and it’s enabled in the OS.
- Distorted audio? Try switching between Intel HDA and AC97.

#### Final Thoughts

Getting audio to work in Proxmox VMs isn’t hard — it just takes the right combo of display type, audio device, and a SPICE client. Once set up, this makes the VM feel much more like a native machine, especially if you’re doing multimedia, video tutorials, or desktop work.

Happy virtualizing!
