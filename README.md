# MT6575/77 Linux support
*Device Tree Source (DTS) files for MediaTek MT6575 and MT6577 Cortex-A9 CPUs for using in mainline Linux.*

## Introduction

Back in 2011~2013, MediaTek MT6575/77 Cortex-A9 CPUs were very popular for low-end mobile phones. There are more than 100 devices utilizing these processors. Moreover, there are so-called tablet revisions (like MT8317) of MT6577 without cellular capabilities. Most of these devices run Android 4.0-4.2, and they have never received any real OS update, althrough there were reports of developers successfully porting Android 4.4. According to my research of public repositories of MediaTek kernels, vast majority of MT6575/77 devices uses Linux kernel v3.4.x. Bringing a mainline support for these processors will give devices a second life.

Yes, these chips were not performant even back in 2012, but they still can power a tiny webserver, a network file storage or some simple digital playground. I encourage everyone who has knowledge to help make it work. I started working on this without having any particular skills and knowledge, and what you see here is result of 6 months of daily hard work.

The code published in this repository has been tested and proved working (more on that below) on Linux v5.8 and Linux v5.9-rc4 running on Acer Iconia B1-A71 (MT8317). The device boots and communicates over UART. **I also made the screen work**, but it uses a framebuffer allocated by UBoot for showing a boot logo. The proper framebuffer+mtkfb binding still had to be made.

---

## What works

1. Generic hardware such as SCU, clocks, timer, sysirq, GIC.

2. 4 UARTs and APDMA. On some MediaTek devices Bluetooth, and WiFi, and other radio communication hardware is controlled over UART. Working UART makes interacting with such hardware easier and provides a room for driver development.

## Important notes and TODOs

(when referring to "Current ...", I mean the latest kernel version at the time of writing this which is 5.9-rc6)

1. **The second core does NOT work on MT6577!!!** The problem must be somewhere in SMP initialization code. Downstream (v3.4.x) kernels for MT6577 have additional code (`mediatek/platform/mt6577/kernel/core/mt-smp.c : void __init smp_init_cpus`) for bringing up the second core (something about software reset workaround), but I failed to port it properly. I decided to keep my attempt to use existing MediaTek SMP initialization system (`arch/arm/mach-mediatek/platsmp.c`) althrough it doesn't work. Nevertheless, first core works fine.

2. It's possible to define watchdog in DTS (`reg = <0xc0000000 0x1c>;`), but **the current binding and driver doesn't support MT6575/77** CPUs. Because of that, the device will power off by itself after a brief period of time without any error or warning. There are 2 workarounds for this:

    2.1 Patch the `mtk_wdt_set_time_out_value` parameter in `mtk_wdt_init` function of your UBoot binary. Keep in mind huge values won't work. If you set the timeout to 255 seconds (0xff), your device will power off little earlier.
    
    2.2 From the `init` script of initramfs, run `devmem 0xC0000000 h 0x2200`. It will effectively turn off the watchdog on hardware level.

3. Current Top Clock Generator **(topckgen) binding and driver doesn't support MT6575/77** CPUs. `reg = <0xc0001000 0x24>;`

4. Current **infracfg binding and driver doesn't support MT6575/77** CPUs. `reg = <0xc0001030 0xed8>;`

5. Current **pericfg binding and driver doesn't support MT6575/77** CPUs, however for some reason kernel boots fine and works great (without any error) if you declare compatibility with MT2701:
```
    pericfg: pericfg@c1000000 {
        compatible = "mediatek,mt2701-pericfg", "syscon";
        reg = <0xc1000000 0x408>;
        #reset-cells = <1>;
        #clock-cells = <1>;
    };
```

6. PMU might not actually work. I have no idea how to check it.

7. Current **I2C driver doesn't support MT6575/77** (but there's a binding) despite MediaTek developer claiming otherwise. First of all, MT6577 I2C bus doesn't use DMA but i2c-mt65xx driver requires it and refuses if no DMA address is specified. Moreover, dt-binding of i2c-mt65xx in the mainline kernel has wrong code for MT6577 example because these values are for MT6589. Apparently, **MediaTek can't** be easily convinced to **fix their own shit** - my attempt at doing so has been replied with '*yadda yadda the name of customer's SOC cannot be accurately evaluated*', more on that on the [linux-i2c mailing list thread](https://marc.info/?l=devicetree&m=159924835523023&w=2). While I have found a [complete datasheet for MT6577](https://cloudflare-ipfs.com/ipfs/QmbpV1joshytPbSGGARwB2pAKjTxfNpWmRB3DJnAXLsq5U) whose values prove I'm right, I can't really mention it on the mailing list because in this case I will be on thin ice since it's sort of proprietary.

## Building

Please keep in mind I wrote this part basing on my experience with MT8317-powered device from 2013. There's a 'acer-b1-a71' directory in this repository where you'll find my kernel configuration and my tiny custom initramfs.

One of the caveats I struggled with is how old MediaTek UBoot loader works. It reads the whole BOOTIMG partition into memory, checks special headers of kernel and initramfs, and boots the kernel if all checks are passes successfully.

I will begin by saying **don't forget to add headers to both kernel and initramfs** every time you are creating a boot.img. There's an utility for this called `mkimage`.

Such an old UBoot does not support DTS out of the box, so a DTS has to be appended to the kernel image (the corresponding kernel config option must be enabled).

Next, every time you make a change in initramfs (.cpio) image, you have to recalculate it's size for `linux,initrd-end` parameter in the device-specific DTS. It's very annoying, to say the least. To combat that, I just embed the compressed initramfs into the kernel image itself. But with this trick, UBoot will refuse to boot because it won't find the initramfs in BOOTIMG. I made an ugly (but working) workaround: even when you have an initramfs already embedded into the kernel image, just add a tiny decoy (dummy) initramfs to boot.img, but don't forget to add a header.

```
# Generate 64 kB of random crap
dd if=/dev/urandom of=ramdisk-dummy bs=64k count=1
# Append a MediaTek header so UBoot will pass a check
mkimage ramdisk-dummy ROOTFS > ramdisk-dummy.mtk 
# Then use it for creating a flashable boot.img:
mkbootimg --kernel ... --ramdisk ramdisk-dummy.mtk -o boot.img
```

A few words on initramfs. I decided to stick with simple busybox-powered initramfs and custom `init` script that will initialize a bare minimal environment and launch a shell on UART. The source is in 'acer-b1-a71' directory. Maybe you'll come up with something better than this! :)

Pro-tip: use .XZ compression for both kernel and ramdisk. It will take a few more seconds to boot, but you'll save a lot of precious space. Just for the sake of example, BOOTIMG partition on my tablet is just 6144 kB.

## But where's a DTS for MT6575?

Since MT6577 is a dual-core version of MT6575, just remove the second CPU core definition from DTS and you're ready. MT6575 also lacks 3rd I2C bus, but as of now it's not declared for MT6577 either.

It might be a great idea to use MT6575 as a base DTS, and create a small DTS for MT6577 with all necessary changes like second CPU core and third I2C bus.

## Post scriptum

I would like to thank everyone in [postmarketOS community](https://wiki.postmarketos.org/wiki/Category:Community) who helped me and expressed any kind of support.
