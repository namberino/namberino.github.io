---
title: "IoT Security Analysis: D-Link DIR-820L Router"
date: 2025-05-31T15:22:55+07:00
toc: true
tags:
  - IoT
  - security
  - reverse engineering
author: "Nam Nguyen"
description: "Security analysis of a D-Link IoT device"
---

## The importance of IoT security

{{< image src="/img/dir820l-security/iot-growth-graph.png" alt="IoT Growth Graph" position="center" style="padding: 10px" >}}

The number of IoT devices in the world has been growing significantly. As more and more devices come online, the more security risks there is. And as we all know, the "S" in "IoT" stands for Security.

The need for security in the IoT sector is growing along with the growth of IoT. Security within IoT is important as there's much bigger consequences if a critical IoT device is hacked compared to a website being hacked. If some IoT devices that's being used for critical purposes such as health monitoring or ICS operation, compromise of such systems could lead to devastating physical consequences. In the worst case, it could endanger human life.

In this blog post, I'll be diving into IoT security analysis. I'll work on the [D-Link DIR-820L](https://legacy.us.dlink.com/pages/product.aspx?id=00c2150966b046b58ba95d8ae3a8f73d) router. Their firmware is available [here](https://legacyfiles.us.dlink.com/DIR-820L/REVA/FIRMWARE/DIR-820L_REVA_FIRMWARE_1.06B02.ZIP).

## Lab environment

I'm analyzing this firmware in Ubuntu 18.04. Here are the tools I'll use.

- [Binwalk](https://github.com/ReFirmLabs/binwalk)
- [Sasquatch](https://github.com/devttys0/sasquatch)
- [QEMU](https://github.com/qemu/qemu)
- [FirmAE](https://github.com/pr0v3rbs/FirmAE)
- [Firmware mod kit](https://github.com/rampageX/firmware-mod-kit)
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra)
- [Burp Suite](https://portswigger.net/burp)

### Binwalk

`binwalk` is used to analyze the firmware and identify the embedded files and file systems within the firmware. It allows us to locate partitions and archives, analyze the entropy of the file, and extract the segments.

### Sasquatch

`sasquatch` is used for unpacking a SquashFS file system. This is basically an improved version of `unsquashfs`.

### QEMU

A full-system emulator and virtualizer that supports a wide range of different hardware architectures. It offers multi-architecture emulation, which is useful for embedded devices, which mostly uses ARM-based or MIPS-based architectures. It also offers networking configurations and debugging support.

### FirmAE

This is a framework that streamlines end-to-end IoT firmware analysis. It automates firmware extraction and unpacking (by utilizing tools like `binwalk` and `sasquatch`). It uses heuristics to get CPU architecture for QEMU, which has been pre-configured. It also setup networking which allows us to access the device's web interface without actually owning the device.

### Firmware mod kit

This tool extracts the underlying file system in the firmware, which allows for modification. This tool can also rebuild the modified firmware into a binary file.

### Ghidra

`ghidra` is a reverse engineering framework with support for multiple different architectures such as x86_64, ARM, MIPS, etc.

### Burp Suite

This is a web security analyzer. It contains many different features that allows analysis of HTTP requests.

## Firmware analysis

Binwalk analysis shows a hidden SquashFS file system in the firmware. There's also extra LZMA compressed data inside the firmware too. The SquashFS file system is commonly used on embedded devices. It's mostly used for the root partition as having it read-only ensure the device can't easily brick itself.

{{< image src="/img/dir820l-security/dir820l-binwalk-results.png" alt="Binwalk result" position="center" style="padding: 10px" >}}

Using the extraction feature of binwalk, we can extract the underlying SquashFS file system.

{{< image src="/img/dir820l-security/dir820l-squashfs-file-system.png" alt="SquashFS file system" position="center" style="padding: 10px" >}}

The `shadow` file contains the password hash of the root user. Running this through `john` gives us the root password, which is `root`, very insecure.

{{< image src="/img/dir820l-security/dir820l-shadow-cracked.png" alt="Cracked root password" position="center" style="padding: 10px" >}}

A quick analysis of the `rcS` startup file shows that the firmware will initialize `bulkListen` and `ncc2` when it starts up. The `ncc2` service is used for processing [Common Gateway Interface (CGI)](https://en.wikipedia.org/wiki/Common_Gateway_Interface) requests. CGI is an interface specification that enables web servers to execute external programs to process HTTP and HTTPS requests. We can find the `ncc2` binary in `/sbin`.

{{< image src="/img/dir820l-security/dir820l-ncc2-sbin.png" alt="sbin/ncc2 file" position="center" style="padding: 10px" >}}

We can also learn that the firmware runs on a 32-bit MIPSEB architecture. Originally I was planning to use IDA for reverse engineering but since IDA Pro is required for analysis of any architecture that isn't x86_64, I opted for Ghidra, which is open source and has support for the MIPS architecture.

Running this binary through `checksec` also shows that this binary doesn't have contain any overflow protection mechanism. It also doesn't contain any debugging symbols, which means it has been stripped, which will make the reverse engineering process a bit harder.

{{< image src="/img/dir820l-security/checksec-ncc2.png" alt="ncc2 checksec" position="center" style="padding: 10px" >}}

## Exploits

I'll cover 4 different exploits for this device.

### Command injection in ping diagnostic

This vulnerability has been identified in the `ncc2` service (`/sbin/ncc2`) of the D-Link DIR-820L. Within the `ncc2` service, there's a functionality called `ping.ccp`, which allows for performing "ping" diagnostics.

{{< image src="/img/dir820l-security/dir-ncc2-ping-check-screen.png" alt="ping check screen" position="center" style="padding: 10px" >}}

Captured request in Burp Suite:

{{< image src="/img/dir820l-security/dir-ping-check-burp-suite.png" alt="ping check request in burp suite" position="center" style="padding: 10px" >}}

The function `FUN_0049e128` is responsible for processing POST requests to `ping.ccp`. This function gets the ping address through the request's `ping_addr` variable.

{{< image src="/img/dir820l-security/cve-2024-51186-request-processing.png" alt="ping request handler" position="center" style="padding: 10px" >}}

The ping address is ran through the `hasInjectionString` function, which was imported from `libleopard.so`. This function checks for the existence of the following 4 characters: "\`", "\\", ";", "'", "|". If any of these characters are found in the user input, the request won't be processed.

{{< image src="/img/dir820l-security/hasinjectionstring-function.png" alt="hasInjectionString function" position="center" style="padding: 10px" >}}

This `hasInjectionString` function did not check for all possible command separators. It is possible to inject a command through the use of the newline character `\n` or `0x0A` in hex.

We can test the command injection by trying to send a GET request to our HTTP server through the router.

{{< image src="/img/dir820l-security/command-injection-dir-ping-cgi-wget-test.png" alt="CMDi ping cgi wget test" position="center" style="padding: 10px" >}}

This works. Now, we can construct an exploit. We can use telnet to create a bind shell using this command.

```sh
telnetd -l /bin/sh -p 9999 -b 0.0.0.0
```

{{< image src="/img/dir820l-security/command-injection-dir-ping-cgi-telnet-bind-shell.png" alt="CMDi ping cgi telnet bind shell" position="center" style="padding: 10px" >}}

To mitigate this vulnerability, the `hasInjectionString` function black list needs to include more command separation characters.

### Stack overflow DoS in cancel ping

The function `FUN_0049e5b0` contains a stack overflow. When the function copies the string from the parameter `nextPage` to `acStack_118`, it uses `strcpy` and doesn't check the length of the string.

{{< image src="/img/dir820l-security/ping-ccp-cancel-ping-function.png" alt="ping ccp cancel ping handler" position="center" style="padding: 10px" >}}

We can test for a buffer overflow DoS by entering a string over 256 characters long. I tested this and found that we need a string with a minimum of 288 characters to successfully DoS the router's `ncc2` service.

{{< image src="/img/dir820l-security/cancelPing-overflow-test.png" alt="cancelPing overflow test" position="center" style="padding: 10px" >}}

To mitigate this, `FUN_0049e5b0` needs to implement some string length check or use `strncpy` for copying strings.

### Firmware modification

An exploit that can be performed if you have access to the physical device is a firmware modification attack. To do this, an attacker need to extract the firmware of the device, unpack the firmware, modify it, then flash the modified firmware on the device. Because the DIR-820L's firmware is available online, we can download it and modify it then flash it on a device later on.

Using Firmware Mod Kit, we can extract the firmware's SquashFS file system into `fmk/rootfs`.

{{< image src="/img/dir820l-security/fmk-rootfs-dir820l.png" alt="fmk/rootfs of firmware" position="center" style="padding: 10px" >}}

Now, we can modify the `rcS` script in `/etc/init.d`, which is used to run additional programs at boot time as specified in `/etc/inittab`. We can modify this file with malicious instructions and execute malicious programs. I'll make it open up a telnet service on port 9999 for now.

{{< image src="/img/dir820l-security/dir820l-rcs-modification-telnet.png" alt="rcS modification with telnet" position="center" style="padding: 10px" >}}

Now, we can build this firmware into a `bin` file.

{{< image src="/img/dir820l-security/dir820-build-new-firmware.png" alt="modified firmware building" position="center" style="padding: 10px" >}}

With the new firmware, we can now flash the firmware onto the router. D-Link DIR routers allow for [emergency flash](http://forums.dlink.com/index.php?topic=44909.0). Since I don't have the device, I'll simulate the modified firmware instead.

{{< image src="/img/dir820l-security/dir820l-modified-fw-telnet.png" alt="modified firmware telnet connection" position="center" style="padding: 10px" >}}

This is the `nmap` scan of the original, unmodified firmware. 

{{< image src="/img/dir820l-security/dir820l-nmap-scan-orig-fw.png" alt="nmap scan original firmware" position="center" style="padding: 10px" >}}

This is the `nmap` scan of the modified firmware. Port 9999 has been opened.

{{< image src="/img/dir820l-security/dir820l-nmap-scan-new-fw.png" alt="nmap scan modified firmware" position="center" style="padding: 10px" >}}

This would only work if the attacker is in the same network as the router. To connect remotely over the Internet, we can insert a backdoor program which can open up a reverse shell to our server.

`netcat` can be used for this. Unfortunately, the `busybox` that we have on the firmware doesn't have `netcat`. So we need to import a MIPSEB `busybox` that has `netcat` into the firmware and use that to create a reverse shell. I used this [`busybox`](https://busybox.net/downloads/binaries/1.21.1/busybox-mips) binary.

I imported the new `busybox` binary into the `/bin` directory and gave it executable permission. Now we can call this `busybox` to use `netcat`.

I made this very simple reverse shell script and stored it in `/bin`. It will check for Internet connection and will only open up a reverse shell if there's an Internet connection. If there is an Internet connection, it will try to establish a `netcat` connection to port 4444 or the specified IP address with `/bin/sh` as the exec file. I'm using my LAN address here as I don't have a server.

{{< image src="/img/dir820l-security/dir820l-backdoor-script.png" alt="backdoor script" position="center" style="padding: 10px" >}}

Now, when we try to emulate this firmware and get `netcat` to listen on port 4444 on our system, we can see that the backdoor works.

{{< image src="/img/dir820l-security/dir820l-backdoor-connection.png" alt="backdoor connection" position="center" style="padding: 10px" >}}

To prevent this attack, the firmware can be encrypted to prevent reverse engineering and unpacking and incorporate TPM (Trusted Platform Module), which provides hardware-based encryption and decryption. The device could also use secure boot with a digitally signed firmware.

### Insecure access control in password change

DIR-820L suffers from insecure access control in the admin account change password functionality of the router. Function `FUN_00451208` has been identified as the function responsible for accepting the CGI request for setting information. 

{{< image src="/img/dir820l-security/ccp-act-set-conditional.png" alt="ccp_act set conditional" position="center" style="padding: 10px" >}}

By default, the password is read from the file `defaultCfg.txt` stored in `sbin`. When the firmware is loaded and executed, the content of the default configuration files are then copied into `/var/tmp/cfg.txt`. Any modification to the configuration done by the admin will be applied to this `cfg.txt` file.

{{< image src="/img/dir820l-security/default-cfg-file-load-function.png" alt="default cfg file load function" position="center" style="padding: 10px" >}}

The `ncc2` program does not perform any validation. This leads to insecure access control. An attacker could craft a packet to the `get_set.ccp` CGI handler without knowing the old password and still be able to change the password of the admin account. The `pure_SetDeviceSettings` function is most likely responsible for handling the admin password change functionality.

{{< image src="/img/dir820l-security/set-device-settings-function.png" alt="set device settings function" position="center" style="padding: 10px" >}}

This function will use `sendEvent` to modify the configuration file in `/var/tmp`. Case 0 of `param_1` will call the `ncc_save_rtcfg` function.

{{< image src="/img/dir820l-security/send-event-load-rtcfg.png" alt="sendEvent load rtcfg" position="center" style="padding: 10px" >}}

There is no checking for old password referenced within `ncc2`. Therefore, we can capture the request to change the password and modify the request to change the password. This request can be launched against other routers remotely.

{{< image src="/img/dir820l-security/password-get-set-ccp-request.png" alt="password get_set.ccp request" position="center" style="padding: 10px" >}}

Sending this request to the router will change the router's configuration file to whatever the request specified. We can check the configuration file read in FirmAE's debug mode.

{{< image src="/img/dir820l-security/var-tmp-cfg-grep-login.png" alt="grep /var/tmp/cfg.txt for login information" position="center" style="padding: 10px" >}}

To mitigate this, the devices needs to implement user session, which would prevent attackers from making unauthenticated requests. Currently, the device only sets a `hasLogin` binary cookie, which is not sufficient for user session validation. Additionally, the device needs to prompt the user for the old password before allowing them to change the password.

## Conclusion

IoT devices are in desperate need of better security. A compromised IoT device could lead to information disclosure in the best case, and death in the worst case. It is imperative that IoT security gets more attention as the consequences could be devastating.

## References

- [IoT Firmware Emulation and Its Security Application in Fuzzing: A Critical Revisit](https://www.mdpi.com/1999-5903/17/1/19)
- [FirmAE: Towards Large-Scale Emulation of IoT Firmware for Dynamic Analysis](https://syssec.kaist.ac.kr/pub/2020/kim_acsac2020.pdf)
- [firmware-mod-kit Documentation](https://code.google.com/archive/p/firmware-mod-kit/wikis/Documentation.wiki)
- [CVE-2024-51186](https://nvd.nist.gov/vuln/detail/CVE-2024-51186)
- [CVE-2023-25281](https://nvd.nist.gov/vuln/detail/CVE-2023-25281)
- [Blended threat](https://en.wikipedia.org/wiki/Blended_threat)
