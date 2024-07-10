---
title: How to test the RAUC D-Bus API locally
---
# Introduction
First of all, this blog does not replace reading the [RAUC Github](https://github.com/rauc/rauc) or [RAUC documentation](https://rauc.readthedocs.io/en/latest/updating.html). If an information in one of those two sources states something that contradicts a part of this blog, you should trust them.

This blog is meant to help you set up RAUC on your local machine (for example on Ubuntu Desktop) so you can test the RAUC D-Bus API, while developing a tool, that is meant to interact with RAUC. Or to integrate it into your End2End tests. This is not covered in the documentation, since the documentation assumes, you will use a Build Tool, like Yocto, what I'm not going to do here. This blog starts at point zero and the last step is a working reponse from the D-Bus API on your terminal.

# Installation
**RAUC Github** differentiates between a Host and a Target. We will use our machine as both. So we will create a bundle on our machine (Host) and we will install things on our machine (Target). We will only install mock files in a mock filesystem, so we don't need a working bootloader.

Following **RAUC Github**, we do the following
```
sudo apt-get install build-essential meson libtool libdbus-1-dev libglib2.0-dev libcurl3-dev libssl-dev libnl-genl-3-dev

git clone https://github.com/rauc/rauc
cd rauc
sudo meson setup build
sudo meson compile -C build
cd build
sudo meson install
```
This is not in RAUC GitHub, but was needed for a working installation:
```
sudo cp /usr/local/share/dbus-1/system.d/de.pengutronix.rauc.conf /usr/share/dbus-1/system.d/
```

# Creating a RAUC bundle
Here we are going to create a RAUC bundle. This bundle will not necessarrily result in a successful installation, but it is going to result in a bundle, that is installable and with it, you can test the D-Bus API.  
In a folder of your choice, do the following:
```
mkdir rootfs
code rootfs/hello.txt (Instead of code, you can use the editor of your choice.)
```
Copy enough text in `hello.txt`, so its bigger than 512 bytes or whatever is needed by RAUC.

``mksquashfs``, which is used now, is installed, when you install RAUC.
```
mkdir bundle
mksquashfs rootfs/ bundle/rootfs.squashfs

openssl req -x509 -newkey rsa:4096 -nodes -keyout demo.key.pem -out demo.cert.pem -subj "/O=rauc Inc./CN=rauc-demo"

code bundle/manifest.raucm
```
manifest.raucm:
```
[update]
compatible=mock
```
Continue:
```
rauc bundle bundle minimal.raucb --cert demo.cert.pem --key demo.key.pem
```

# Further setting up RAUC
Here we create a custom mock bootloader, that is able to handle the requests of RAUC.
```
code bootloader.sh
```
bootloader.sh:
```sh
#! /bin/sh

case $1 in
  get-current)
      echo "system0" # this has to match with system.conf, which we will create later in the guide
      ;;
  get-primary)
      echo "system0" # this has to match with system.conf, which we will create later in the guide
      ;;
  set-primary)
      exit 1
      ;;
  get-state)
      echo "good"
      ;;
  set-state)
      exit 1
      ;;
  *)
      echo "unknown option $1"
      exit 1
      ;;
esac
exit 0
```
Make it executable:
```
chmod +x /path/to/bootloader.sh 
```
We set up a loop device using this [Guide](https://linuxhandbook.com/create-virtual-block-device/), which can be used by RAUC:
```
dd if=/dev/zero of=Mock.img bs=1M count=2
sudo losetup -fP ./Mock.img
sudo mkfs.ext4 ./Mock.img
mkdir partition
```
Look up the number of your loop device. For me its loop0:
```
sudo losetup -a
```
```
sudo mount -o loop /dev/loop0 ./partition
```

```
sudo mkdir /etc/rauc
sudo micro /etc/rauc/system.conf (Again, you don't have to use micro, but can use the editor of your choice)
```
system.conf ("system0" in here has to match with the name in bootloader.sh):
```
[system]
compatible=mock
bootloader=custom

[keyring]
path=/path/to/demo.cert.pem

[slot.rootfs.0]
device=/path/to/partition
type=raw
bootname=system0 

[handlers]
bootloader-custom-backend=/path/to/bootloader.sh
```
Now you can start the service and the API works:
```
sudo rauc service
dbus-send --system --dest=de.pengutronix.rauc --type=method_call --print-reply / de.pengutronix.rauc.Installer.GetSlotStatus
```