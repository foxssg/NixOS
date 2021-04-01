This is a NixOs tutorial
Thu Nov 28 14:47:03 -03 2019
I'm writing this because I haven't encountered a easy 2019 tutorial, I've used a 2015 tutorial to start with nixos but it was full of useless commands and there was wrong information which complicated the installation a bit, I've used this tutorial:
https://chris-martin.org/2015/installing-nixos


My setup is:
Lenovo x220 with a brand new SSD

1- Download the .iso from the official page: https://nixos.org/nixos/download.html (I'm using version 19.09)
2- Make a booteable USB (you can do it with linux terminal, search for "dd")
3- When you boot ir for the first time and select the first option you will be granted acces to the command line, if you want you can use "systemctl start display-manager" to enter de GUI
4- You have to partitioning your disk, if you're doing a fresh install with only one hard drive, you can do the following (this is my example, yours could be not be identical):

gdisk /dev/sda
n
(enter)
(enter)
+1000KiB
EF02

n
(enter)
(enter)
+512MiB
EF00

n
(enter)
(enter)
(enter)
8E00

w


5- You have to open the terminal and type:
cryptsetup luksFormat /dev/sda3
cryptsetup luksOpen /dev/sda3 enc-pv

6- Create LVM

pvcreate /dev/mapper/enc-pv

vgcreate vg /dev/mapper/enc-pv

lvcreate -n swap vg -L 10G

lvcreate -n root vg -l 100%FREE



7- Format partition

mkfs.vfat -n BOOT /dev/sda2

mkfs.ext4 -L root /dev/vg/root

mkswap -L swap /dev/vg/swap




8- Mount it

mount /dev/vg/root /mnt

mkdir /mnt/boot

mount /dev/sda2 /mnt/boot




9- Swap

swapon /dev/vg/swap




10- Configuration

nixos-generate-config --root /mnt




11- Go to /mnt/etc/nixo and add the following to configuration.nix:

boot.initrd.luks.devices = [
  {
    name = "root";
    device = "/dev/sda3";
    preLVM = true;
  }
];

boot.loader.grub.device = "/dev/sda";

networking.wireless.enable = true;


12- This is the stage that I had problem with, wifi. This tutorial was my salvation:
https://shapeshed.com/linux-wifi/

As a root create a file named "example.conf" in /etc/wpa_supplicant and in that file write:

ctrl_interface=DIR=/run/wpa_supplicant GROUP=wheel

Then run this command:

wpa_supplicant -B -i wlp3s0 -c /etc/wpa_supplicant/example.conf

Be aware it's possible that "wlp3s0" wouldn't work in your configuration, check that name with "ifconfig"

If you get "Successfully initialized wpa_supplicant"  you hace to run wpa_cli command then type "scan" then "scan_results" then "add_network" then "set_network 0 "network_name" then "set_network 0 psk "network_password" then "save config"



13- Type nixos-install and if everything went well you're almost done (if something failed, check typos), then reboot.



14- To enable graphical environment add this to /etc/nixos/configuration.nix :

services.xserver = {
  enable = true;
  desktopManager.kde4.enable = true;
  synaptics.enable = true;
};

15- To add an user add this to configuration.nix (then use passwd command to create one) :

users.extraUsers.USERNIX = {
  name = "USERNIX";
  group = "users";
  extraGroups = [
    "wheel" "disk" "audio" "video"
    "networkmanager" "systemd-journal"
  ];
  createHome = true;
  uid = 1000;
  home = "/home/USERNIX";
  shell = "/run/current-system/sw/bin/bash";
};



16- To enable network manager replace:

networking.wireless.enable = true;

with

networking.networkmanager.enable = true;


and add plasma5.networkmanager to the package list

