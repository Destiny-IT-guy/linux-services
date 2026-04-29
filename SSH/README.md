# SSH

SSH (secure shell) is one of the most important protocol when working with linux systems. It solves a huge problem with telnet. Telnet who uses TCP port 23 allows you to connect to a system remotely but the whole connection is unencrypted. SSH fixes that security risk by encrypting the connection. SSH uses by default port <b>22</b> and is used in most linux system and most network equipment.</br>

SSH works by a client trying to establish a remote connection to a remote server. They establish a tcp connection between each other but that isn't enough. They have to agree on an encryption algorithm then perform a key exchange to obtain a shared secret. Then the server needs to prove its identity to protect against malicious actors. After all of this the client can authenticate itself using 2 different ways.</br></br> Before we go deep in linux system i just want to mention that SSH is installed on most Windows systems. by default only openssh client is enable so you can ssh from a windows devices to a linux system but not in reverse. You can enable openssh server on windows. Some other protocols like RDP exist for a full remote connection Windows systems.

## Openssh

Openssh is the standard app uses for ssh connection on linux system. There exist other smaller sized apps but most systems comes with this installed by default. To install it use the following command.

```bash
sudo apt update
sudo apt install openssh-client && sudo apt install openssh-server
ssh -V
sudo systemctl start ssh
sudo systemctl status ssh
```

To simply ssh into a machine use the following command.

```bash
ssh <username>@<IP or hostname>
```

You should read the ssh manual to see more advance options. If you have configured dns the hostname might be a easier way.

### SSH keys

As you know we can use simple password authentication to get remote access to a system. There is an alternative way where more used in professional environment. SSH keys are a asymmetric cryptography used to authenticate users. IT is very useful because it is more secure than password authentication and simpler to use after setting them up.

To start off create your ssh keys. I will create ssh keys with 4096 bits and name them test. the first command make a key in the directory where you are located at the second will create the keys in the specified folder.

```bash
ssh-keygen -t rsa -b 4096
ssh-keygen -t rsa -b 4096 -f ~/.ssh/test
ssh-keygen -t ed25519  -f ~/.ssh/test #recommended modern key
```

two keys should be created. my keys are named `test` which is the private key and `test.pub` which is the public key. Then i recommend you move both file to `~/.ssh/` folder. After this we can copy our public key to the remote system authorized keys folder with the following command.

```bash
ssh-copy-id -i ~/.ssh/test.pub <remote username>@<remote IP>
```

After this you would normally be able to ssh to the remote machine without needing to authenticate yourself but since we didn't use the default name for the ssh keys we will hav to modify our ssh configuration.


here exist to different ssh configuration files. `ssh_config` for ssh client configuration and `sshd_config` for the ssh server configuration

### <u>config and ssh_config</u>

Theses files are organized by sections. These files location are at `~/.ssh/config` or `/etc/ssh/ssh_config`. The first one is a user level file. It might not be created by default but will have higher <b>priority</b> over the one in `etc`. To create it use the following command.

```bash
mkdir -p ~/.ssh
touch ~/.ssh/config
chmod 600 ~/.ssh/config
```

Each section is delimited by the word `Host`. Here are some example on how to use host.

```bash
Host 192.168.2.1 # single host
Host 192.168.2.* # wildcard match
Host server.victim.net # specific hostname
Host *.victim.net # every hostname that ends with .victim.net
Host 192.168.2.1 192.168.2.200 10.10.10.10 # multiple host with same rules
Host * #default for every host
```

There exist other option you can use like `?` or `!` but the ones I mentioned should be enough. For a more detailed explanation go to `man ssh_config`.</br>

During college I had to use a couple of option to solve some problem I encountered. Some cisco ios require to use older and weaker encryption algorithm which are disabled by default in ssh_config. To fix this problem I added these options.

```bash
Host 192.168.2.50 10.10.10.10
    KexAlgorithms +diffie-hellman-group1-sha1
    Ciphers +aes128-cbc,3des-cbc,aes192-cbc,aes256-cbc
    HostKeyAlgorithms +ssh-rsa,ssh-dss
    PubkeyAcceptedKeyTypes +ssh-rsa,ssh-dss
    StrictHostKeyChecking accept-new

Host *
    ...
```

The `+` in front of some option is used to add those current option to a list of default. Without it let say for ciphers, It would deleted more secure ciphers and only used the ones i configured.</br></br>
The other of which you put your section is <b>Very</b> important. These rules apply from top down. Let's say im trying to ssh to an IP 192.168.2.117. what port will it use.

```bash
host 192.168.2.117
    port 2222

Host 192.168.2.*
    port 2000

Host *
    # port 22
```

In this case it will use port 2222. If I change the order and put 192.168.2.* on top the port 2000 would be chosen even if a more specific option appears in the file. Tho any option use while using the `ssh` command will <b>override</b> rules defined in system or user configuration files. Here is what i mean visually.

| location | priority |
|----------|----------|
| /etc/ssh/ssh_config | lowest |
| ~/.ssh/config | higher |
| command line | highest |

</br> Now let's get into our ssh keys problem. Since we used custom keys name ssh won't try to use are private key to try and connect to the remote host. To solve this we will need to add the key location in our configuration file. Let say i have two sets of keys one for special network equipments and other for everything else. This is what my configuration would look like.

```bash
Host 192.168.2.50 10.10.10.10
    IdentityFile ~/.ssh/test

Host *
    ...
```

Since my public key its on those 2 device my private key is the only one that can validate by signing the authentication request.

### ssh keys with network equipment

I will do this another day. both ways.

### <u>sshd_config</u>

I will complete this tomorrow