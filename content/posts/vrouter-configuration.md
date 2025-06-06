+++
date = '2025-06-05T19:23:03+02:00'
title = 'LANCOM vRouter Config'
+++

This short guide is about how to configure a [LANCOM vRouter](https://www.lancom-systems.de/produkte/router-sd-wan/central-site-vpn-gateways/lancom-vrouter). I haven't found any proper documentation on how to do it via the CLI, so someone might find this useful.

This guide will cover the following setps:
1. an IP-Address for it's main INTRANET network
2. a user with admin rights for configuration via LANconfig
3. change the second interface, to be used as WAN-Uplink

### Prequesites
LANCON allows you to install thier vRouter, with the LAN-Ports being limited to 1 Mbit/s, until the device is licensed. There is a free 30-day trial [available here](https://my.lancom-systems.de/service-support/registrierungen/demo-lizenzen/).
After following the basic [Installation Guide](https://www.lancom-systems.de/download/LC-vRouter/IG_vRouter_DE.pdf) for your desired platform, make sure to add an additional LAN-Port, which will function as the WAN-Uplink.

### Step 1: Set IP Address

Navigate to the TCP/IP network list and assign an IP address to the standard INTRANET network:
```bash
cd /Setup/TCP-IP/Network-list
set INTRANET 10.0.99.254 255.255.255.0
```

### Step 2: Create a User

Navigate to the admin configuration directory and create a new user:

```bash
cd /Setup/Config/Admins/
add lc-user
```

Change to the newly created user and set a Password:


```bash
cd lc-user
set Password Secret1!
```

Assign Access Rights with the following command:

```bash
set Access-Rights su
```

After completing this step, the user user is configured with supervisor rights and the specified password.

### Step 3: Assign the WAN-Interface

As the last step you need to add the second physical interface to the same virtual network as your INTRANET network.

Navigate to the second Ethernet port and assign it to the virtual network interface LAN-1:

```bash
cd /Setup/Interfaces/Ethernet-Ports/ETH-2
ls
set Assignment LAN-1
```


You can now use LANconfig to further configure the vRouter.