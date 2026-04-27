# NTP

Network time protocol (NTP) is a protocol used to synchronize clocks between computers servers and equipment over a network. Without it security logs might be out of sync and jobs that require syncing with multiple system might fail.

## how it works

Before we start NTP use port 123 and uses UDP to ensure low latency. NTP is organizes system in a hierarchy named Stratum. The lowers stratum numbers level are closer to the main clock and higher number are farther. Lower level receive time information from higher levels. Here is a basic table explaining Stratum levels.

| Stratum level | description |
|---------------|-------------|
| 0 | A references clock with high levels of precision. Usually atomic clocks or GPS receivers |
| 1 | System linked to stratum 0 devices |
| 2 | System that gets its time from a stratum 1 device |
| 3 | System that gets its time from a stratum 2 device |

### Clients synchronization

The client and server ntp process works by an exchange of packets between both to synchronize the clients clock. To determine the right time the client will use a total of four timestamps.

<b>1.</b> Client sends ntp request to its server with his own time stamps. <b>T1</b>  </br>
<b>2.</b> Server records the time when he receive T1: <b>T2</b> </br>
<b>3.</b>The server records the time when he sending reply to client: <b>T3 </b>  </br>
<b>4.</b>Client records the times when he receive T3 : <b>T4</b>  </br>


After receiving all 4 timestamp the client will do 2 operations to determine the time.

<b> Round trip delay :</b> (T4-T1) - (T3 - T2)</br>
This calculate the network delay.

<b> Clock offset : </b> [(T2 - T1) + (T3 - T4)] /2 </br>
This calculate difference between client times and server times.

### Adjustment

With both numbers, the system can adjust its time. In most case it will gradually speed up the time or slow its time to perfectly align it with the server time which is called (Slewing). If the time difference is to big it might make a big change right away also called (Stepping). The client will continuously send ntp request to restart the process but the intervals depends on its time sync ith the server.

## Configuration

We will start by updating our packages and install ntpsec 
```bash
sudo apt update
sudo apt install ntpsec
```

Now We will go to NTPsec configuration file `/etc/ntpsec/ntp.conf`  to modify our service

Here is what a typical file would have

```bash
tos maxclock 11
tos minclock 4 minsane 3
pool 0.ca.pool.ntp.org iburst
pool 1.ca.pool.ntp.org iburst
pool 2.ca.pool.ntp.org iburst
pool 3.ca.pool.ntp.org iburst
# Use Ubuntu's ntp server as a fallback.
server ntp.ubuntu.com
#restrict default kod nomodify noquery limited
#restrict -6 default kod nomodify noquery limited
restrict default ignore
restrict -6 default ignore
restrict source nomodify notrap noquery

restrict 192.168.2.0 mask 255.255.255.0 nomodify notrap noquery
restrict 10.10.10.0 mask 255.255.255.0 nomodify notrap noquery
restrict 10.10.20.0 mask 255.255.255.0 nomodify notrap noquery
restrict 10.10.30.0 mask 255.255.255.0 nomodify notrap noquery
restrict 10.10.100.0 mask 255.255.255.0 nomodify notrap noquery
restrict 127.0.0.1
restrict ::1
```

By default some pool will already be defined. I went on the following site [ntpool.org](https://www.ntppool.org/en/) to get my canadian pools. it list how many pools are available on each continent and each country. To view all available option configuration you can go to ntpsec official site [ntpsec.org](https://docs.ntpsec.org/latest/ntp_conf.html) but here is a short list of hat i used

The `iburst` option help with the initial synchronization. Sends multiple rapid request instead of 1. There is also a burst option available which sends multiple request every time the server is polled.

Now lets get into the security parts of my commands.</br>

The <b><u>default</u></b> option is use to apply the rules to everyone including ipv6 with the `-6`. </br>
The <b><u>ignore</u></b> option reject every incoming packet even if my server sent a request first. Do not use `default ignore` by itself or client and the server wont be able to synchronize.</br>
The <b><u>source</u></b> option solves my ignore problem.It's a rule that applies to IP address my server learns dynamically from the pools or servers.<br>

The <b><u>kod</u></b> option tells clients to stop sending request if they are sending too many. </br>
The <b><u>nomodify</u></b> option removes client permission to change this server ntp configuration. </br></br>

After adding your commands to the configuration file its time to start ntpsec.

```bash
sudo systemctl start ntpsec
sudo systemctl enable ntpsec
sudo systemctl status ntpsec
```

## Testing

The easiest way to test if your serving is communicating with the pools you set up is to use `ntpq` command. You can go read it's manual to see more advance option but `ntpq -p` should be enough to assured you that your synchronization is healthy.

The `*` in front of a server means that your server is synced with it.</br>
the `+` in front of many server indicates its a backup choice to sync with.</br>
Lastly if you encounter any other problem observer the st(stratum), reach and offset columns.

## Clients

There are multiple ways to connect client to your ntpsec server. You can install ntpsec on your clients, install Chorny on you clients or even use systemd timesyncd.

### timesyncd

modify this configuration file `/etc/systemd/timesyncd.conf` so it looks something like this.

```bash
[Time]
NTP=<IP address>
FallbackNTP=ntp.ubuntu.com
```

Then restart the service and check if it worked.

```bash
sudo systemctl restart systemd-timesyncd
timedatectl timesync-status
date
```

### Chrony

First we need to install th service

```bash
sudo apt update
sudo apt install chrony
```

Modify th configuration file `/etc/chrony/chrony.conf` so it looks something like this.

```bash
#pool ntp.ubuntu.com        iburst maxsources 4
#pool 0.ubuntu.pool.ntp.org iburst maxsources 1
#pool 1.ubuntu.pool.ntp.org iburst maxsources 1
#pool 2.ubuntu.pool.ntp.org iburst maxsources 2
server 192.168.2.232 iburst
server ntp.ubuntu.com iburst
```

Restart the service and test.

```bash
sudo systemctl restart chrony
chronyc sources -v
```

# NTS

As you know ntp isn't secure and all traffic it create is unencrypted/unauthenticated. This is a vulnerability that can be a problem in professional environment. To solve this NTS(network time security) was created. IT adds authentication to ensure the packets you receive comes by the desired servers by using NTS-KE to establish secure session keys. For the keys exchange NTS uses tcp and port <b>4460</b>. Afterwards it uses ntp regular port for time synchronization.

Where to deploy NTS? You can deploy only on your edge side so you ony uses NTS to get time from the internet and your internal network continue to use NTP or you can use NTS for both.

## NTS configuration

To get time from the internet securely you will have to add public NTS servers and add them to your configuration files. Here are 2 NTS servers i added.

```bash
server time.cloudflare.com iburst nts prefer
server nts.netnod.se iburst nts prefer
```

The <b><u>prefer</u></b> option select one or sources to a have higher priority than all other sources. also if you have multiple prefer's like me the first to appear in your configuration will take priority even if it has a worse connection so make sure your best NTS connection is on top.

To ensure it works and that its using NTS instead of NTP I like to use 2 commands. the first one is to ensure that im able to establish a tcp connection with the designated NTS server and the seconds is to look thru the logs.

```bash
nc -zv time.cloudflare.com 4460
sudo journalctl -u ntpsec | grep -i NTS
```

## complete NTS
