# Packer plugin for ARM images

This plugin lets you take an existing ARM image, and modify it on your x86 machine.
It is optimized for raspberry pi use case - MBR partition table, with the file system partition 
being the last partition.

With this plugin, you can:
- Provision new ARM images from existing ones.
- Use ARM binaries for provisioning ('apt-get install' for example)
- Resize the last partition (the filesystem partition in the raspberry pi) in case you need more
  space than the default.

Tested for raspbian images on built on Ubuntu 17.10. It is based partly on the chroot aws 
provisioner, though the code was copied to prevent aws dependenceies.

# How it works?

The plugin runs the provisioners in a chroot envrionment. binary execution is done using
qemu-arm-static, via binfmt_misc.


## Dependencies:
This builder uses the following shell commands:
- kpartx - mapping the partitons to mountable devices
- qemu-user-static - Executing arm binaries

To install the needed binaries:
```
sudo apt install kpartx qemu-user-static
```
Other commands that are used are (that should already be installed) : mount, umount, cp, ls, chroot.

To resize the filesystem, the following commands are used:
- e2fsck
- resize2fs

To provide custom arguments to `qemu-arm-static` using the `qemu_args` config, `gcc` is required (to compile a C wrapper).

Note: resizing is only supported for the last active
partition in an MBR partition table (as there is no need to move things).

This builder uses the following uses this kernel feature:
- support for `/proc/sys/fs/binfmt_misc` so that ARM binaries are automatically executed with qemu

## Operation
This provisioner allows you to run packer provisioners on your ARM image locally. It does so by mounting the image on to the local file system, and then using `chroot` combined with `binfmt_misc` to the provisioners in a simulated ARM environment.

# Configuration
To use, you need to provide an existing image that we will then modify. We re-use packer's support 
for downloading ISOs (though the image should not be an ISO file).
Supporting also zipped images (enabling you downloading official raspbian images directly).

See [example.json](example.json) and [builder.go](pkg/builder/builder.go) for details.

# Compiling and Testing
## Building
As this tool performs low-level OS manipulations - consider using a vm to run this code for isolation. While this is highly recommended, it is not manadatory.

This project uses [go modules](https://github.com/golang/go/wiki/Modules) for dependencies introduced in Go 1.11.
To build:
```bash
git clone https://github.com/solo-io/packer-builder-arm-image
cd packer-builder-arm-image
go mod download
go build
```

## Running with Vagrant
This project includes a Vagrant file and helper script that build a VM runtime environment. The runtime environment has 
custom provisions to build an image in an iterative fashion (thanks to @tommie-lie for adding this feature).

To use the Vagrant environment, run the following commands:

```
git clone https://github.com/solo-io/packer-builder-arm-image
cd packer-builder-arm-image
vagrant up
```

To build an image edit example.json (or set PACKERFILE to point to your json config), and use `vagrant provision` like so :
```
vagrant provision --provision-with build-image
```
The example config produces an image with go installed and extends the filesystem by 1GB.

That's it! Flash it and run!
# Flashing

We have a post-processor stage for flashing. You can also use the command line:
```
go build cmd/flasher/main.go
```

It will auto-detect most things and guides you with questions.

# Cookbook
# Raspberry Pi

(see full examples in contrib folder)
Add these provisioners to:

## Eanble ssh
```json
{
  "type": "shell",
  "inline": ["touch /boot/ssh"]
}
```
## Set wifi password
setet user variables name wifi_name and wifi_password. then:

```json
    {
      "type": "shell",
      "inline": [
        "echo 'network={' >> /etc/wpa_supplicant/wpa_supplicant.conf",
        "echo '    ssid=\"{{user `wifi_name`}}\"' >> /etc/wpa_supplicant/wpa_supplicant.conf",
        "echo '    psk=\"{{user `wifi_password`}}\"' >> /etc/wpa_supplicant/wpa_supplicant.conf",
        "echo '}' >> /etc/wpa_supplicant/wpa_supplicant.conf"
        ]
    }
```

## Add ssh key to authorized keys, enable ssh, disable password login.
This example locks down the image to only use your 
current ssh key. Disabling password login makes it extra secure for networked environments. Note:
this example requires you to run the plugin without a VM, as it copies your local ssh key.

```json
{
  "variables": {
    "ssh_key_src": "{{env `HOME`}}/.ssh/id_rsa.pub",
    "image_home_dir": "/home/pi"
  },
  "builders": [
    {
    "type": "arm-image",
    "iso_url" : "https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2017-12-01/2017-11-29-raspbian-stretch-lite.zip",
    "iso_checksum_type":"sha256",
    "iso_checksum":"e942b70072f2e83c446b9de6f202eb8f9692c06e7d92c343361340cc016e0c9f",
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline": [
        "mkdir {{user `image_home_dir`}}/.ssh"
      ]
    },
    {
      "type": "file",
      "source": "{{user `ssh_key_src`}}",
      "destination": "{{user `image_home_dir`}}/.ssh/authorized_keys"
    },
    {
      "type": "shell",
      "inline": [
        "touch /boot/ssh"
      ]
    },
    {
      "type": "shell",
      "inline": [
        "sed '/PasswordAuthentication/d' -i /etc/ssh/ssh_config",
        "echo  >> /etc/ssh/ssh_config",
        "echo 'PasswordAuthentication no' >> /etc/ssh/ssh_config"
      ]
    }
  ]
}
```

## A complete example:
See everything included in here: [contrib/pi-secure-wifi-ssh.json](contrib/pi-secure-wifi-ssh.json). Build like so:
```
sudo packer -var wifi_name=SSID -var wifi_password=PASSWORD build contrib/pi-secure-wifi-ssh.json
# or  if running from vagrant ssh:
sudo packer build  -var wifi_name=SSID  -var wifi_password=PASSWORD /vagrant/contrib/pi-secure-wifi-ssh.json
```


