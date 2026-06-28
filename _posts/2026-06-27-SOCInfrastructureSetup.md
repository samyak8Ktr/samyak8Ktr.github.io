---
title: "ECN Lab 4: Setting up a Wazuh SIEM, then investigating Windows, Linux and  Suricata alerts."
date: 2026-06-28
tags:
  - SecurityEngineering
  - SOC
  - Wazuh
  - SIEM
  - threat-hunting
categories: cybersecurity_engineering
image: /images/SL4d.png
---


## INTRODUCTION

**SCENARIO:** We are appointed as security engineer to secure a virtual company named "samyak.corp". Currently there have 50+ employees in the company and there are no security measures, no network management, no AD, nothing just all connected to a LAN. 

Welcome to the fourth installment in my **"Enterprise Cyber Security and Networking Lab" (ECN Lab)** series.

So far we had:
- Isolated the WAN network from Clients using a firewall server and a switch.
- Designed a HA firewall fail-over architecture (for the sake on simplification I will remove that from here).
- Designed a multi WAN fail-over and load balancing architecture,

Now, the owner of `samyak.corp` had told they plan to establish a SOC team on the premises.

We will setup a Wazuh SIEM server and install wazuh agents on the client devices. Including both windows and Linux machines.

Lastly I will add a wazuh agent on Opnsense Firewall to see the Intrusion detection logs.

![](images/ECN4.png)

---
## SOME CHANGES FOR HOME LAB

(If someone is following me along)

I had been using virtual-box for the past 2 setups:
- firewall HA setup
- Multi WAN failover setup

Reason was that in firewall HA setup, vmware workstation's anti-MAC spoofing security was messing with the virtual IP's functioning.

**But now** for SOC lab I might have to run 6 - 7 VMs together. Personally, I consider VMware more performant and a more familiar software.

So, I will switch to **VMware** from now on.

And for the sake of performance, I would remove the HA or multi-WAN architecture.

---
## NETWORK

LAB network | vmware network | IP range | DHCP
LAN | custom network | 10.0.0.0/24 | disabled
WAN | NAT network | 192.168.81.0/24 | enabled

---
## SOME MACHINES TO SETUP

We will need:
- OpnSense Firewall (10.0.0.251)
- Ubuntu Server (10.0.0.134)
- Windows 10 pro (10.0.0.135)
- Wazuh VM (Cent OS) (10.0.0.4)
- Kali linux (10.0.0.133)

> Turn off VM which are not required in testing a particular feature
{: .prompt-tip }

---
## WAZUH SERVER SETUP

**NOTE:** Wazuh has three components:
1. Wazuh Server
2. Wazuh Indexer 
3. Wazuh dashboard

All these can be deployed across multi-node cluster configuration. Great for large enterprise environments.

But, for less than 100 endpoints there is an all in one deployment. See [here](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html)

Install the wazuh VM, documentation says it needs 8 GB memory and 4 cores, for lab 4 gb ram and 2 cores worked fine.

just connect it to our internal network (LAN network: 10.0.0.0/24)

Use the credentials `wazuh-user:wazuh`
![](images/Pasted image 20260626232841.png)

Now we need this VM to use static IPv4 say **10.0.0.8/24** . Problem is it's preconfigured to use DHCP, and not have nmcli installed.

We will need to edit the configuration file directly

```sh
sudo vim /etc/sysconfig/network-scripts/ifcfg-<interface_name>
```

> Use ifconfig to see the LAN interface name
{: .prompt-info }

```
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPADDR=10.0.0.8
PREFIX=24
# or NETMASK=255.255.255.0
GATEWAY=10.0.0.254
DNS1=8.8.8.8          
USERCTL=yes
PEERDNS=no            # Don't overwrite DNS via DHCP
```

remove this lines as we are not using DHCP.
```
DHCPV6C=yes
DHCPV6C_OPTIONS="-nw"
PERSISTENT_DHCLIENT=yes
RES_OPTIONS="timeout:2 attempts:5"
```

restart the network service
```sh
sudo systemctl restart network
```

do some checks:
```
ifconfig interface_name
ping -c 4 8.8.8.8 
```

go to `https://wazuh_vm_ip` , use default creds `admin:admin`
![](images/Pasted image 20260626235852.png)

![](images/Pasted image 20260627000118.png)

### TIME DATE ISSUE

For me the timestamps of the logs were messed up, tuns out the **wazuh server time was incorrect**.

In my lab it was fixed by syncing with vmware.

```sh
[wazuh-user@wazuh-server ~]$ vmware-toolbox-cmd timesync enable 
Enabled 

[wazuh-user@wazuh-server ~]$ timedatectl 
Local time: Sun 2026-06-28 00:32:26 IST 
Universal time: Sat 2026-06-27 19:02:26 UTC 
RTC time: Sat 2026-06-27 19:02:26 
Time zone: Asia/Kolkata (IST, +0530) 
System clock synchronized: yes 
NTP service: active RTC in local TZ: no
```

---
## DEPLOYING WAZUH AGENTS ON HOSTS

Go to **Deploy new agent** select the OS , Wazuh server IP (so the agent can connect to it).

### WINDOWS

Download ISO from official Microsoft website and install, if you already haven't

![](images/Pasted image 20260627140541.png)

Administrator powershell:
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='10.0.0.4' WAZUH_AGENT_NAME='windows10_host_agent'

NET start Wazuh
```

Verify the service is runing
![](/images/Pasted image 20260627140859.png)
### UBUNTU SERVER 

Download the latest ISO and go through the installation:
- Enable SSH for remote access.
- give at-least 2 GB ram.
- Connect to LAN network
- Update the server.

Assign a static IP:

find a file like `/etc/netplan/00-installer-config.yaml`:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 10.0.0.134/24
      routes:
        - to: default
          via: 10.0.0.251
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

```sh
sudo netplan try # accept the configurations if okay
# test 
ip a s 
ip r
```

SSH from windows host to Ubuntu server:

```sh
ssh user1@10.0.0.134
```

Adding the web proxy certificate for HTTPS

download **.crt** on windows, and use SCP to transfer it to linux:
```powershell
scp OPNsense-webCA.crt user1@10.0.0.134:/home/user1
```

install it on ubuntu server:
```sh
sudo cp OPNsense-webCA.crt /usr/local/share/ca-certificates/OPNsense-webCA.crt
sudo update-ca-certificates
```

Enter the generated command, for my case:
```sh
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb && sudo WAZUH_MANAGER='10.0.0.4' WAZUH_AGENT_NAME='ubuntu_server_agent' dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
```

If you still encounter any errors, check [here](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html)

See that the daemons are running:
![](images/Pasted image 20260627144924.png)
### VERIFYING DEPLOYMENT

Both are agents on active:
![](images/Pasted image 20260627144958.png)


---

## EXPLORING WAZUH
If wazuh finds any system vulnerability or compliance issues it shows them. 
![](images/Pasted image 20260627210802.png)

As, I am using am old version of windows 10
![](images/Pasted image 20260627210844.png)

---

## RULES

All the out-of-the-box rules are placed in `/var/ossec/ruleset`. Read more about them [here](https://documentation.wazuh.com/current/user-manual/ruleset/index.html).

```
/var/ossec/
        ├─ etc/
        │   ├─ decoders/
        |   |        └─ local_decoder.xml
        │   └─ rules/
        |         └─ local_rules.xml
        └─ ruleset/
                ├─ decoders/
                └─ rules/
```

![](images/Pasted image 20260628001552.png)

We can also find them in `Server Management > Rules`

![](images/Pasted image 20260628004625.png)

---

## SEEING THOSE ALERTS

On the **Discover** tab we can see and filter all the alerts. 

### WAZUH SERVER ITSELF

Yes, an **agent 000** is installed on wazuh server itself. A SIEM is also a high profile target for attackers with access to it they can hide all thier activities

So we should also know how to monitor the SIEM itself.

viewing active connections on wazuh server, an enumeration technique by attackers after gaining initail access.

```
[wazuh-user@wazuh-server ~]$ sudo netstat -tlnp
```

See the alert shows the command executed.

![](images/Pasted image 20260628010816.png)


### UBUNTU

The interface is too cluttered.

We will add some required field from the sidebar, I selected some:

![](/images/Pasted image 20260628011214.png)

First time executing the `sudo` command after a login

```sh
user1@ubuntserver:~$ sudo su
```

Sort with timestamp and filter for agent 002. 

```
agent.id:002
```

See the log showing sudo was executed.

![](/images/Pasted image 20260628011851.png)

### WINDOWS

We will create a user account **wazuhtest** and add it to the Administrators group.

Open an administrator powershell / CMD:

```
net user wazuhtest P@ssw0rd123! /add
net localgroup Administrators wazuhtest /add
```

filter for agent 001 and set the timeline accordingly 

```
agent.id:001
```

See the alerts showing up.

![](images/Pasted image 20260628100244.png)

We can see that **wazuhtest** user was created

![](images/Pasted image 20260628100148.png)

Alert for adding the **wazuhtest** user to the **administrator** group.

![](images/Pasted image 20260628100900.png)

---

## DEPLOYING WAZUH AGENT ON SURICATA

Check the Opnsense Documentation [here](https://documentation.wazuh.com/4.14/installation-guide/wazuh-agent/index.html).

We need to install the plugin `os-wazuh-agent` to Opnsense. 

Find the Wazuh service and point the agent to our wazuh server (10.0.0.4). And atleast select **suricata** from all the options.

![](images/Pasted image 20260628104703.png)

See it in wazuh. ( For no performance issues, I have disabled the other two machines for now )

![](images/Pasted image 20260628104748.png)

### TESTING

In the first post of these series, I made a very trivial suricata rule to detect **SYN port scan**. I will use the same for testing here.

From the kali machine:
```sh
sudo nmap -sS 10.0.0.251 -Pn --top-ports 500    
```

See, Opnsense Sending IDS events to wazuh server.

![](images/Pasted image 20260628105406.png)

Inspect the alerts:

![](images/Pasted image 20260628105629.png)

---

## CONCLUSION

With this the fourth installment of **"Enterprise cybersecurity and networking lab"** 

In next lab, I will add various threat intel tools to the SIEM. So they can aid analysts in the investigations.

---
