---
layout: post
title: Access Your Modem Through OpenWrt Router
categories: [Network,Linux]
excerpt: How to access your bridged modem through an OpenWrt router
---
# Introduction
This guide goes through how to configure your OpenWrt router to allow access to the web interface of your bridged xDSL modem.

## Prerequisites
- A router with OpenWrt installed
- LuCI web interface
- A bridged xDSL modem
- You will need a seperate subnet for access to the modem. In this guide I am using the following:
  - 192.168.10.0/24 for the LAN subnet
  - 192.168.20.0/24 for the modem's subnet

# Create Interface
1. Sign into LuCI web interface on your OpenWrt router.
2. Highlight Network and click on Interfaces
