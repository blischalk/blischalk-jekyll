---
layout: post
title: "Kankun Smart Plug Analysis"
description: "During the Offensive Internet Of Things course, the Kankun Smart Plug is analyzed in various ways including: using Jadx to decompile and analyze the mobile app, acquiring and analyzing the device's firmware, and a nalyzing the network traffic."
tags: [firmware, analysis, kankun, android]
---

This blog post has been created for completing the requirements of the
SecurityTube Offensive Internet of Things course.

[Offensive Internet Of Things Exploitation](http://www.securitytube-training.com/online-courses/offensive-internet-of-things-exploitation/index.html)

Student ID: IoTE- 778

## Overview

During the Offensive Internet Of Things course the
[Kankun Smart Plug](https://www.amazon.com/Kankun-Socket-Remote-iPhone-Smartphone/dp/B00N8N5NNK):
![Kankun Smart Plug](/assets/Kankun/AmazonKankunSmartPlug.png)
is analyzed in various ways including:

- Using Jadx to decompile and analyze the mobile app
- Acquiring and analyzing the device's firmware
- Analyzing the network traffic
- Utilizing
  [Kankun Controller](https://github.com/0x00string/kankuncontroller)
  to perform various attacks on the device

My goal with this post is to:

- Replicate the techniques outlined in the course
- Document the process and any gotchas encountered along the way

## Analyzing The Android App with Unzip and Jadx

The first thing covered which I wanted to attempt was decompiling the
Android applications with
[Jadx](https://github.com/skylot/jadx). Prior to taking this course I
didn't realize it was possible to decompile an Android application
back into `.java` files. I was under the impression that the best Java
decompiling you could get was to `.class` files. After a bit of
investigation, this seems to be because Android applications are
actually `.dex` (Davlik Executable Files) zipped into `.apk`
archives. This is something I've made a mental note to revisit and
learn more about.

After looking in the manual that came with the Kankun Smart Plug, it
indicates that you can scan a QR code to get the Android or iOS apps,
or you can download the Android `.apk` from here:
http://kk.huafeng.com:8081/none/android/Smartwifi.apk

With a little wget magic we have out Android App:

    wget http://kk.huafeng.com:8081/none/android/Smartwifi.apk

After a `brew install jadx`, `jadx Smartwifi.apk`, and `unzip
Smartwifi.apk` we have an un-archived Android application as well as a
decompiled version of the Android application. Once this is done, a
few interesting things are noticed:

	$ ls lib/armeabi/
	libNDK_03.id0  libNDK_03.id1  libNDK_03.nam  libNDK_03.so

It appears that there is a "Shared Object File", analogous to a
Windows DLL that can be analyzed. This would seem to indicate that the
application makes some native calls. This could be interesting.

Inside of the decompiled Android app, the following application
structure exists:

	$ ls Smartwifi
	AndroidManifest.xml  adapt/               cache/               custom/              res/
	Smartwifi.iml        android/             com/                 hangzhou/

Looking at the application structure seems to indicate that the
"hangzhou" directory most likely contains the application
namespaces. Upon further investigtion we find:

	$ ls Smartwifi/hangzhou/
	kankun/ zx/

	$ ls Smartwifi/hangzhou/kankun/
	AlertUtil.java               DeviceTaskActivity.java      NumericWheelAdapter.java     ShowDialogActivity.java      WheelView.java
	ArrayWheelAdapter.java       GetElectricityService.java   OnWheelChangedListener.java  SmartwifiActivity.java       WifiAdmin.java
	Config.java                  GuideActivity.java           OnWheelScrollListener.java   UpdateModel.java             WifiJniC.java
	ControlHelpActivity.java     JudgeDate.java               ProtectService.java          ViewPagerAdapter.java        dbHelper.java
	DBManager.java               NetStateUtil.java            ScreenInfo.java              WheelAdapter.java
	DeviceActivity.java          NetworkTool.java             SelectPicPopupWindow.java    WheelMain.java

	$ ls Smartwifi/hangzhou/zx/
	BuildConfig.java      PreferencesUtil.java  R.java

In fact this does seem to be where the application namespaces are located.

## Calling Strings on the Shared Object

Circling back to `libNDK_03.so` which was shown earlier, we can run
the `strings` command on it to see if there are any interesting
strings found. Lets try:

    strings libNDK_03.so

	DecryptData
	Java_hangzhou_kankun_WifiJniC_decode
	EncryptData
	Java_hangzhou_kankun_WifiJniC_encode

	fdsl;mewrjope456fds4fbvfnjwaugfo <--------- Very Interesting.........

	aes_set_key
	aes_encrypt
	aes_decrypt

Using strings on the shared object yields some interesting finds. We learn that:

- The Android app does in fact ues JNI to interface with the shared object
- The library appears to be using aes encryption
- We find a string that is 32 bytes long or 8 * 32 = 256 bits long...

Since we have a 256 bit string along with the fact that the Android
app is performing aes encryption, it is very likely that the 256 bit
string is actually the AES-256 bit encryption key. We'll hold on to
that for later...

## Android App Source Code Review

Reviewing the souce code we find some intesesting things:

{% highlight java %}

package hangzhou.zx;

.. snip ..

public class PreferencesUtil {
    public static String filePathString = "http://app.jkonke.com/kkeps.bin";

{% endhighlight %}

The `PreferencesUtil` file seems to indicate where the firmware can be
acquired for firmware analysis. We'll explore that later.

In `WifiJniC.java` we find the function definitions for calling the
native code in the shared object:


{% highlight java %}

package hangzhou.kankun;

public class WifiJniC {
    private static final String libSoName = "NDK_03";

    public native int add(int i, int i2);

    public native String codeMethod(String str);

    public native String decode(byte[] bArr, int i);

    public native byte[] encode(String str, int i);

    static {
        System.loadLibrary(libSoName);
    }
}

{% endhighlight %}

Looking at the API that the shared library seems to provide, it would
seem that if we look for the encode / decode function calls within the
application, that we can get an idea of the types of messages that the
Android App sends and receives from the smartplug. Within
`DeviceActivity.java` we find various incantations similar to the following:

{% highlight java %}

.. snip ..

while (!getAck) {
  try {
	address = InetAddress.getByName(DeviceActivity.this.dev_ip);
	udp_cmd = "lan_phone%" + DeviceActivity.this.dev_mac + "%" + DeviceActivity.this.dev_word + "%" + this.sendcmd;
	cmd_buf = DeviceActivity.this.jnic.encode(udp_cmd, udp_cmd.length());
	DeviceActivity.this.cmdSocket.send(new DatagramPacket(cmd_buf, cmd_buf.length, address, Integer.valueOf(DeviceActivity.this.dev_port).intValue()));
  } catch (IOException e4) {
    e4.printStackTrace();
  }

.. snip ..

{% endhighlight %}

With this we can glean the following information about the messages
that are used for communication:

- It appears that the app communicates to the device over UDP
- There is a prefix of `lan_phone` or `wan_phone`. This most likely
  has to do with whether the device is setup to be a wireless access
  point on its own, or whether it is setup as a device connected to a
  router on the wireless network.
- Followed by the mac address of the smartplug
- Followed by a password maybe???
- Followed by a command
- Sometimes followed by an argument for the command


## Summary

At this point we have gathered a good bit of information about how
this device works and perhaps some secrets that could let us have some
fun analysing the network traffic. The course mentions a tool for
manipulating the Kankun smartplug called:
[Kankuncontroller](https://github.com/0x00string/kankuncontroller) I
haven't had a chance to play around with it yet but looking at the
source code shows that the author has figured out all kinds of ways to
play with the smart plug and get it to do things without having to use
the mobile app.

This post is getting a bit long so I think this is a good point to
stop and I'll make a part 2 covering analysis of the network traffic
as well as analysis of the firmware.
