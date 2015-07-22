---
layout: post
title:  "Using USB-to-Serial adapter on Mac OSX"
description: Using USB-to-Serial adapter on Mac OSX.
permalink: /using-usb-to-serial-adapter-on-mac-osx
---
First you need to install USB-to-Serial adapter driver for PL2303. [Click here for PL2303 USB-to-Serial Driver](http://www.prolific.com.tw/US/supportDownload.aspx?FileType=56&FileID=133&pcid=85&Page=0). Username and password is `guest`
After installing the correct driver, plug in your USB-to-Serial adapter, and open a Terminal session 
Enter the command `ls /dev/cu.*`, and look for something like usbserial.
{% highlight sh %}    
air:~ hafidz$ ls /dev/cu.*
/dev/cu.Bluetooth-Incoming-Port
/dev/cu.usbserial
{% endhighlight %} 
Then type `screen /dev/cu.usbserial 9600` to access serial console.
To quit the screen app, type `CTRL-A`, then `CTRL-\`.