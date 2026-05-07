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

As I said before, this configuration file is used to defined rules for incoming ssh request. Like the other, this file has a manuel (`man sshd_config`) but I will show and briefly explain some basics option you might want to use.


```bash
# Enable legacy support for Cisco IOS
KexAlgorithms +diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
Ciphers +aes128-cbc,aes256-cbc,3des-cbc
HostKeyAlgorithms +ssh-rsa
CASignatureAlgorithms +ssh-rsa
PubkeyAcceptedAlgorithms +ssh-rsa

banner /etc/issue.net # Banner message for ssh connection

# Access
PermitRootLogin prohibit-password
AllowUsers test root toto@192.168.2.231 titi # allow users and allow users from specific IP
AllowGroups admin # when allow users and groups exists users must be in both to obtain access
# AllowGroups or users with no option locks out everyone
DenyUsers toto@192.168.1.* # deny ssh connection to toto from 192.168.1.X subnet
#DenyGroups also exist.

# Authentication
PermitEmptyPasswords no
PubkeyAuthentication yes
PasswordAuthentication yes
MaxAuthTries 2

# Network options
Port 2222 
addressfamily any # inet4 for ipv4, inet6 for ipv6 or any for both
ListenAddress 192.168.2.232 # binds ssh to a single IP if your server has many IP address.

# rules
LoginGraceTime 45 # Time in seconds for user to successfully authenticate before disconnection
ClientAliveInterval 5m # how long of no activity for server to send message to client
ClientAliveCountMax 3 # how many unanswered it take for ssh session to be terminated.

ChannelTimeout direct-tcpip=5m session=10m # disconnect port forwarding  session if no data has been transferred for 5 min and regular ssh session if no user activity for 10 minutes.

# Additional security
MaxSessions 5 # limits how many session ssh multiplexing can use 
MaxStartups 10:50:40 # limit how many unauthenticated connection. It protects against brute force attacks. allows 10 unauth connection, after that any more it will drop 50 percent of connection. when it reaches 40 it drops all unauth connection.
IgnoreRhosts yes


# LOG
#SyslogFacility AUTHPRIV
#LogLevel VERBOSE
```

Here is some common rules I might to add to my sshd config file. After modifying the file restart ssh to apply these options.

## SSH tunneling

SSH tunneling redirect network traffic through a encrypted ssh session. It allows users to access privates services that are behind firewall more securely. But they can cause a big security vulnerability in a network if not properly used. I will briefly show 3 types of tunneling but others exist like agent forwarding and X11 forwarding.

### <u>local port forwarding</u>

local port forwarding works by redirecting traffic from a single port on your local machine thru ssh session to access a specif port on the other machine. Let say I want to access a server web page but its firewall only allows incoming traffic from its ssh port (2222). I would use the following command. 

```bash
ssh -L 8008:localhost:443 <username>@<IP> -p 2222
```

The first port is where my local traffic will go. The localhost right after is the localhost of the machine im going to ssh into. the following port is the service im trying to access.</br>
With this i will be able to access my remote server web interface via my https://localhost:8008.</br></br>Let say I want to access my pfsense web interface but my IP is blocked but my remote linux server is allowed. This is the command I will use.

```bash
ssh -L 8080:<Pfsense-IP>:443 <username>@<IP> -p 2222
```

With this my remote server will act as a relay between pfsense and my local machine. My local host traffic on my port 8080 will go thru my remote server ssh port 2222 to then finally arrives to pfsense web interface, the packets it receive will see the remote linux server as senders IP address instead of my local machine. To access it via a browser I would use the  url https://localhost:8080.

### <u>Reverse port forwarding</u>

This can be considered as the opposite of local port forwarding. It allows remote servers to access a service on my local machine that might usually might not be accessible because its behind a firewall/NAT. Let say im running a database on my local machine and I want a remote server on the cloud or behind a public IP/Nat that is running prometheus/grafana to have access to that service. If the remote server is reachable and ssh enable I will be able to create a tunnel between my machine and remote server. Here is an example of how to use the command.

```bash
# ssh -R <Remote_port>:localhost:<local_port> <username>@<IP>
ssh -R 8080:localhost:3601 <username>@<IP> -p 2222
```

This will forward traffic from remote server port 8080 to the ssh tunnel to my local machine 3601 port. The localhost in the command refers to my local machine here instead of remote servers localhost. To access the SQL service
the remote server would use `localhost:8080 or 127.0.0.1:8080`.</br>
Like -L option, the reverse port forwarding can act as a relay for another service running on another machine on my local network. Let say my local machine is .2.200 but my sql service is running on .2.250. This would be the command I use.

```bash
ssh -R 8080:192.168.2.250:3601 <username>@<IP> -p 2222
```

### <u>Dynamic port forwarding</u>




## ssh multiplexing
