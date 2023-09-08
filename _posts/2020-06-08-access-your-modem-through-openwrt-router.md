---
title: Access Your Modem Through OpenWrt Router
categories: [Guides,OpenWrt]
tags: [openwrt,guides,modem,network,firewall]
---

This guide goes through how to configure your OpenWrt router to allow access to the web interface of your bridged xDSL modem.

## Prerequisites
- A router with OpenWrt 19.07 installed
- LuCI web interface
- A bridged xDSL modem
- You will need a seperate subnet for access to the modem. In this guide I am using the following:
  - 192.168.10.0/24 for the LAN subnet
  - 192.168.1.0/24 for the modem's subnet

# Create Interface
1. Sign into LuCI web interface on your OpenWrt router
2. Highlight **Network** and click on **Interfaces**
3. At the bottom left click on **Add New Interface...**
4. Enter a name for the interface and select the wan switch VLAN interface from the drop down menu
![](/assets/img/2020-06-08-access-your-modem-through-openwrt-router/interface.png)
5. Click on Create Interface
6. Enter an IP address in the range of your modem. In this example the modem's IP is 192.168.1.1
    - This means you will have to set the IP of your modem interface to an IP between 192.168.1.2 and 254
7. Click on the drop down menu next to **IPv4 netmask** and select 255.255.255.0
![](/assets/img/2020-06-08-access-your-modem-through-openwrt-router/general.png)
8. Click on the Firewall tab
9. In the **Create / Assign firewall-zone** drop down menu select the wan zone
![](/assets/img/2020-06-08-access-your-modem-through-openwrt-router/firewall.png)
10. Click on Save & Apply

You should now be able to browse to your modem's web console.
