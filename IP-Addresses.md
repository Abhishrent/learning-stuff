# IP addresses: the internet's postal system

---

Before two computers exchange a single byte, they need to answer one question: where are you? The IP address is that answer. Think of it as a postal address, except nobody writes it on an envelope and the postman is made of electrical signals.

## IPv4 and IPv6: a numbers problem

IPv4 uses 32 bits split into four octets. Each octet is 8 bits, so each one holds a value from 0 to 255. That gives you addresses like `192.168.1.1` and rules out nonsense like `192.300.1.1`.

32 bits = roughly 4.3 billion addresses. Sounds like a lot until you realise there are more connected devices on earth than people. We ran out. IPv6 fixes this with 128 bits and enough addresses to give every grain of sand on earth its own IP and still have leftovers.

## Public vs private: two worlds, one router

Your laptop and your phone both connect to the same WiFi. They have different private IPs but share one public IP — your router's. Three address ranges are permanently reserved for private networks and never appear on the public internet.

```
10.0.0.0    – 10.255.255.255
172.16.0.0  – 172.31.255.255
192.168.0.0 – 192.168.255.255
```

Everything outside these ranges is a public IP. Your router is the bouncer standing between your private network and the internet.

## NAT: one public IP, many devices

How does your router know which device a response belongs to? It keeps a NAT table. Every time your laptop makes a request, the router logs it. When the response comes back, the router checks the table and delivers it to the right device.

The catch: if nobody on your network initiated the connection, the router has no table entry and drops the packet. This is why running a home server requires port forwarding. Port forwarding tells your router to always send incoming traffic on a specific port to a specific private IP on your network. Port forwarding tells your router to always send incoming traffic on a specific port to a specific private IP on your network. You manually tell the router "anything coming in on port 3000, always send it to my laptop at `192.168.1.2`." That is a permanent table entry.

## DHCP: the automatic address vending machine

Your router runs a DHCP (Dynamic Host Configuration Protocol) server that hands out private IPs automatically whenever a device joins the network. Leases expire and IPs can change, which is a problem if you have a port forwarding rule pointing at a specific private IP. The fix is a DHCP reservation: you tell the router to always give your machine the same IP based on its MAC address.

Your ISP does the exact same thing for your router's public IP. Unless you pay extra for a static public IP, it can change anytime.

## 127.0.0.1: the address that never leaves your machine

`127.0.0.1`, aka `localhost`, always means your own computer. Traffic to this address never touches your network. When you run an Express server and visit `http://localhost:3000`, the request bounces off your own machine without going anywhere.

This mapping lives in `/etc/hosts`, a file your OS checks before making any DNS request. You can add your own entries. Add `127.0.0.1 petuhost` and suddenly `http://petuhost:3000` works exactly like localhost. Fun party trick, genuinely useful for simulating production domains locally.

## Binding in Express: who can reach your server?

When you call `app.listen()` you decide which network interface accepts connections.

| Call | Who can connect |
|------|-----------------|
| `app.listen(3000)` | Everyone (all interfaces) |
| `app.listen(3000, '0.0.0.0')` | Everyone (explicit) |
| `app.listen(3000, '127.0.0.1')` | Only your own machine |

In production, always use `0.0.0.0`. Binding to `127.0.0.1` on a deployed server means you built an app nobody can reach. Classic.

## DNS: the internet's phonebook

Nobody memorises IP addresses. DNS (Domain Name System) translates `google.com` into `216.58.203.46` so you do not have to. When you type a URL, your computer resolves it in order:

```
1. Local hosts file (/etc/hosts)
2. Local DNS cache
3. External DNS server (e.g. 8.8.8.8)
```

Your domain registrar hosts the DNS records that point your domain at your server's IP. If your public IP is dynamic, a DDNS service keeps that record updated automatically so your domain does not go stale.

## The localhost trap

You hardcode `http://localhost:3000` in your React frontend during development. It works perfectly. You deploy. Everything breaks. Why? Because `localhost` in a user's browser points to their own machine, not yours.

The fix is one line and an environment variable.

```js
const BASE_URL = process.env.REACT_APP_API_URL
```

Set it to `http://localhost:3000` in development and your deployed backend URL in production. Same code, right behavior everywhere.


