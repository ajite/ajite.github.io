---
layout: post
title: Raspberry Pi with KODI that works with Netflix and other streaming services behind a proxy
date:   2021-06-30 10:05:28 +0800
categories: GFW bucardo postgres
excerpt: A set up to run streaming services such as Netflix and Disney Plus on a RasperryPi with Kodi behind a proxy. This article focuses on the proxy set up and make sure that all the traffic go through it.
---

# By pass geo-blocking warning 

Most of the streaming services forbid the use of proxy/VPN. This article it meant to help you set up a box in a complex home environment. You should not use it to access geo-locked content.

I tested this setup with Netflix, Hbo Max, Disney Plus and YouTube. Everything worked like a charm.
This article focuses on teaching you how to install the proxy client and make sure that all the traffic goes through it.


# Introduction

Before starting the setup make sure of three things:

1. Have an access to a Shadowsocks proxy. You may setup like [Outline](https://getoutline.org/).
2. The proxied network must have access to Netflix, Disney Plus or Hbo Max.
3. You do have a streaming service account

This was tested on a Rasperry Pi 3B+. It might not work with earlier version.

# Operating System.

I recommend using OSMC TV. LibreElec seems great but it is very difficult to install all the packages we need. You can download the latest image [here](https://osmc.tv/download/).

Copy the image into your SD card with your favourite tool. I used Raspberry PI Imager.
As usual you place the SD card into the PI, boot the PI and follow the instructions on the screen to set up the network and enable the SSH.

Tips: the default user and password are both "osmc"

Once logged in SSH get root access.

```bash
sudo -s
```

# Install and configure the Shadowsocks client

```bash
apt update
apt install -y shadowsocks-libev
```

Configure the client credentials.

```bash
vi /etc/shadowsocks-libev/config.json 
```

You can check if it works by using two terminals.

One one terminal:

```bash
ss-local -c /etc/shadowsocks-libev/config.json 
```

On the other terminal you can type that to check if the IP is correct:

```bash
curl --socks5 127.0.0.1:1080 http://ifconfig.me
```

Then it's important to make it start on boot.

You can do that by editing /etc/rc.local

Make sure the file ends like below:

```
ss-local -c /etc/shadowsocks-libev/config.json
exit 0
```

Make sure it is excutable.

```bash
chmod +x /etc/rc.local
```

Then create or edit /etc/systemd/system/rc-local.service and paste the config below.

```
[Unit]
    Description=/etc/rc.local Compatibility
    ConditionPathExists=/etc/rc.local

[Service]
    Type=forking
    ExecStart=/etc/rc.local start
    TimeoutSec=0
    StandardOutput=tty
    RemainAfterExit=yes

[Install]
    WantedBy=multi-user.target
```

It is a good time to reboot it and check if it works.

# Install Polipo

OpenElec is using Python 2.7. Netflix and other add-ons will not be able to query natively through the Socks5 proxy.
There are python libraries that you can set up through pip that support it but then having an HTTP proxy sounded better.
That is why you will need to install Polipo and set it up to use the previous Socks5 proxy as a parent.

```bash
apt-get install -y polipo
```

Add these lines in /etc/polipo/config

```
socksParentProxy = "127.0.0.1:1080" 
socksProxyType = socks5
proxyAddress = "127.0.0.1"
proxyPort = 54321 
```

Restart it:

/etc/init.d/polipo restart

Check that you are getting the expected IP address:

```bash
curl --proxy 127.0.0.1:54321 http://ifconfig.me
```

# Install Redsocks

The last step to finish our setup is to install Redsocks and configure it.
We want to make sure that all the requests go through the proxy.

```bash
apt-get install -y redsocks
```

I found it easier to change the iptables using:

```bash
update-alternatives --config iptables
```

Select: /usr/sbin/iptables-legacy


I shamelessly stole the following rules from the internet (Reddit or Stack Overflow). You can copy and paste them. You might need to modify them if your private network is not specified in the rule set.

```bash
iptables -t nat -N REDSOCKS

# Do not forget to add your private network if it is not listed here.
iptables -t nat -A REDSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A REDSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A REDSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A REDSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A REDSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A REDSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A REDSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A REDSOCKS -d 240.0.0.0/4 -j RETURN
iptables -t nat -A REDSOCKS -d 38.143.2.214 -j RETURN
iptables -t nat -A REDSOCKS -d 192.168.0.0/24 -j RETURN
iptables -t nat -A REDSOCKS -d 192.168.1.0/24 -j RETURN
iptables -t nat -A REDSOCKS -p tcp --dport 1080 -j RETURN
iptables -t nat -A REDSOCKS -p tcp -j REDIRECT --to-ports 12345

iptables -t nat -A REDSOCKS -d 0.0.0.0/8 -j RETURN
iptables -t nat -A REDSOCKS -d 10.0.0.0/8 -j RETURN
iptables -t nat -A REDSOCKS -d 100.64.0.0/10 -j RETURN
iptables -t nat -A REDSOCKS -d 127.0.0.0/8 -j RETURN
iptables -t nat -A REDSOCKS -d 169.254.0.0/16 -j RETURN
iptables -t nat -A REDSOCKS -d 172.16.0.0/12 -j RETURN
iptables -t nat -A REDSOCKS -d 192.168.0.0/16 -j RETURN
iptables -t nat -A REDSOCKS -d 192.168.0.0/24 -j RETURN
iptables -t nat -A REDSOCKS -d 192.168.1.0/24 -j RETURN
iptables -t nat -A REDSOCKS -d 198.18.0.0/15 -j RETURN
iptables -t nat -A REDSOCKS -d 224.0.0.0/4 -j RETURN
iptables -t nat -A REDSOCKS -d 240.0.0.0/4 -j RETURN
iptables -t nat -A REDSOCKS -d 240.0.0.0/4 -j RETURN
iptables -t nat -A REDSOCKS -d 38.143.2.214 -j RETURN

# Anything else should be redirected to port 1080
iptables -t nat -A REDSOCKS -p tcp -j REDIRECT --to-ports 

# Any tcp connection made by osmc and root should be redirected.
iptables -t nat -A OUTPUT -p tcp -m owner --uid-owner osmc -j REDSOCKS
iptables -t nat -A OUTPUT -p tcp -m owner --uid-owner root -j REDSOCKS
```

Once again make sure it works.

```bash
curl ifconfig.me
```

The only things left to do is to make the rules persistent.
The easiest way is to install “iptables-persistent” and save the rules when asked by the instasller.

```bash
apt-get install -y iptables-persistent
```

# Install Streaming services

You have plenty of tutorials on the internet that teach you how to do that.
This section is just to give you some tips that I deemed relevant for Netflix.

## Netflix

Before installing Netflix make sure you run as the osmc user the following commands:

```bash
sudo apt -y install python-pip python-setuptools build-essential
# Run that as the osmc user, not root.
pip install --user pycryptodomex
```

Then go to this [Github repo](https://github.com/CastagnaIT/plugin.video.netflix/) and follow the instructions for KODI 18.x LEIA

If you are having a delay between the sound and the image try to limit the video to 720p or 1080p.
Make sure you are using a proper power supply.

## Hbo Max, Disney Plus

I used the [slyguy repo](https://k.slyguy.xyz/)

## YouTube

Kodi has a default YouTube addon you can install. You will need to create an API Key.
[Click here](https://kodi.wiki/view/Add-on:YouTube) for more info.

I have been using this setup for a month and it works like a charm.
