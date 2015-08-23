---
layout: post
title:  "Menukar lokasi lalai VirtualBox Machine"
description: Bagaimana caranya untuk kita menukar lokasi lalai VirtualBox Machine.
permalink: /menukar-lokasi-lalai-virtualbox
---
Sunting fail **`/Users/username/Library/VirtualBox/VirtualBox.xml`**

Pada 
{% highlight xml %}
<SystemProperties defaultMachineFolder="/Users/username/VirtualBox VMs" defaultHardDiskFormat="VDI" VRDEAuthLibrary="VBoxAuth" webServiceAuthLibrary="VBoxAuth" LogHistoryCount="3" exclusiveHwVirt="false"/>
{% endhighlight %}
Tukar defaultMachineFolder kepada
{% highlight xml %}
<SystemProperties defaultMachineFolder="/Volumes/ExternelDrive/VirtualBox VMs" defaultHardDiskFormat="VDI" VRDEAuthLibrary="VBoxAuth" webServiceAuthLibrary="VBoxAuth" LogHistoryCount="3" exclusiveHwVirt="false"/>
{% endhighlight %}
