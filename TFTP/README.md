# TFTP

Trivia file transfer protocol is a simple protocol used for transferring files between client and server. TFTP use udp and port 69. IT is a simpler protocol compared to FTP and it also uses less bandwidth. TFTP doesn't u'se authentication so it is also less secure. Because of this it shouldn't only be used in you own local network and you shouldn't make it available to the internet with port forwarding.

## use cases

There are a couple of use cases that TFTP excel at. It is great for booting devices in a network. We can use TFTP to make a physical computer download and configure operating system without any intervention needed. I use it when i use wds/mdt and fog. IT can also be used to push firmware update for network devices. I also use it to push configuration file to networking device. let say i deleted a iso for my switch i can use tftp to put it back on the device.

## installation

A good tftp to use is tftpd-hpa. Use tge following command to install it.

```bash
sudo apt update
sudo apt install tftpd-hpa
```

make a folder to store tftp files.

```bash
sudo mkdir -p /srv/tftp
sudo chmod -R 777 /srv/tftp
```

The tftp configuration file location is in `/etc/default/tftpd-hpa`

Here is a basic configuration.

```bash
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure --create"
```

Username is the user who runs the tftp service.

Directory is folder where you upload or download files from.

address is the ip address and port tftp listen on. If you put no IP address it will listen on all available interface on your server.

secure restrict tftp to the tftp directory only.

create allow user to upload file to the server.

After changing your configuration file restart tftp and make sure th service is running.

```bash
systemctl restart tftpd-hpa
systemctl status tftpd-hpa
```

## download switch image/config

I will show how to download your configuration file from your network switch.

first look in the flash to know what file you want to download.

```cisco
enable
dir flash:
```

locate the file you want to copy. When writing the command make sure to enter the file full name.  If you don't want to change the destination file name press enter else write the name you want.

```cisco
copy flash:config.text tftp:
Address or name of remote host []? <IP ADDRESS>
Destination filename [config.text]?
!!
1083 bytes copied in 0.23 secs
Switch#
```

## Tftp client

```bash
apt install tftp
apt install atftp
```

To download config.text use the following configuration use the following for tftp and atftp.

```bash
tftp <IP ADDRESS>
tftp> get config.text
tftp> quit

atftp <IP ADDRESS>
tftp> get config.text
tftp> quit
```