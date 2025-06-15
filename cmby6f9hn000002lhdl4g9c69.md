---
title: "Taming the RX 7800 XT: Clean GPU Reset in Windows VMs Without a Reboot (QEMU)"
datePublished: Sun Jun 15 2025 21:27:35 GMT+0000 (Coordinated Universal Time)
cuid: cmby6f9hn000002lhdl4g9c69
slug: taming-the-rx-7800-xt-clean-gpu-reset-in-windows-vms-without-a-reboot-qemu
tags: ai, virtual-machine, amd, gaming, fix, proxmox, inference, passthrough

---

# Intro

I’ve recently recased my PC into a Fractal Mood, in my opinion its a really stunning looking enclosure for my components, and it blends in nicely with my overall office setup. Anyway, that part is kinda irrelevant in most ways except one. Instead of opting to continue using Windows 11 on this computer I decided now was the time to make my move to Linux.

I have not been using Windows in any serious manner for a little while now, we use MacOS at work and deploy onto GCP which is primarily a linux environment. I’ve been looking for an excuse to reduce the amount of Windows in my life.

Occasionally I do game so I can’t get away from it completely, I’m aware of the absolute awesomeness that is Proton (thanks Gabe!) but there are still cases where I need Windows to get my gaming fill.

I was already aware of Proxmox, having used it in the past so I decided I really like the idea of turning my PC into more of a homelab underpinned by Proxmox goodness. My plan was to have a Windows VM with my RX 7800 XT passed through into it when required but also wanted to be able to shut that down and switch it over to a Linux VM for some light inference tasks.

# The Problem

I was able to relatively easily setup the Windows guest with GPU passthrough, managed to install the operating system and also managed to pass my GPU through no problem. The issues started when I created a second Linux VM that I planned to run Ollama on. When I tried to shutdown the Windows VM and then start the Linux, the host would just go nope and essentially crash. I had to reboot it to get back to where I needed to be.

I attempted many, many, many things to try and get to the behaviour that I wanted. One observation I did make was that the Linux guest (Fedora to be precise) released the graphics card properly. So if I shutdown the Linux guest and than booted up the Windows one, it worked fine. Not a problem the GPU switching to Windows. If I then shutdown Windows and tried to boot the Linux guest, it crashed. So I came to the conclusion that the problem actually lay with Windows.

Another observation I made, if I “stopped” the VM, as opposed to “shutdown” the GPU would release ok, even from Windows. But I wanted ACPI shutdown to avoid any data issues. It turns out, it’s Windows that wasn’t releasing the card properly.

Some of the things I tried without success:

* Installed QEMU drivers on guest
    
* Installed QEMU agent
    
* Attempted to pass the VBIOS into the VM via a rom file
    
* Installed latest AMD drivers
    

None of these things worked

# The Fix

There is a utility in Windows called pnputil which can be used to interact with hardware. It can also be used to disable hardware via a command line. Using this tool I was able to get the Windows VM to shut down gracefully and release the GPU. The steps I followed were

* in CMD (elevated) run “pnputil /enum-devices /class Display?
    
    * This will print out a load of information and your output may vary but look for the GPU you are interested in
        
    * ```plaintext
        Instance ID:                PCI\VEN_1002&DEV_747E&SUBSYS_2427148C&REV_C8\4&23f93f77&0&00E0
        Device Description:         AMD Radeon RX 7800 XT
        Class Name:                 Display
        Class GUID:                 {4d36e968-e325-11ce-bfc1-08002be10318}
        Manufacturer Name:          Advanced Micro Devices, Inc.
        Status:                     Started
        Driver Name:                oem4.inf
        ```
        
        You want to make note of the Instance ID
        
* type “pnputil /disable-device "PCI\\VEN\_1002&DEV\_747E&SUBSYS\_2427148C&REV\_C8\\4&23f93f77&0&00E0" and you should get some positive response.
    
* **Congratulations** you can now shutdown the VM and release the GPU gracefully
    

# Automate it

Doing the above manually all the time would have been a trouble, but luckily we can easily automate this via gpedit.msc.

go here:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1750022609117/ff1ed13d-5711-4a66-876d-6e3c0ec3f825.png align="center")

create some scripts somewhere on your C: drive

## enable-gpu.bat

Create the below script, double click on Startup and browse to where you saved it

```plaintext
@echo off
pnputil /enable-device "PCI\VEN_1002&DEV_747E&SUBSYS_2427148C&REV_C8\4&23f93f77&0&00E0"
```

## disable-gpu.bat

Same as above but for Shutdown

```plaintext
@echo off
pnputil /disable-device "PCI\VEN_1002&DEV_747E&SUBSYS_2427148C&REV_C8\4&23f93f77&0&00E0"
```

# Conclusion

Do the above and you have the ability to shutdown and release your AMD GPU (tested with 7800xt) from a Windows guest, mega right?