---
layout: post
title: "Intercepting Flutter Traffic: Bypassing Proxy Issues with Dnsmasq and Burp Suite"
date: 2026-03-17
tags: [Flutter, Pentesting, Burp Suite, Dnsmasq, Mobile Security, Interception]
comments: true
published: false
---

If you've ever tried to intercept a Flutter application, you've likely hit a wall. Unlike most native apps, Flutter (specifically the Dart `HttpClient`) often ignores system-level proxy settings. This makes traditional proxying through Burp Suite or Charles Proxy incredibly frustrating for security researchers and developers alike.

But there's a foolproof way to force Flutter to talk to your proxy: **DNS Spoofing**.

In this guide, we'll walk through how to use `dnsmasq` and Burp Suite to intercept Flutter traffic by tricking the application into thinking your proxy machine is the actual API server.

---

## The Problem: Why Flutter Ignores Proxies

Flutter uses its own networking stack (Dart IO). While it _can_ support proxies, many apps are built without explicit proxy support in the code, and the underlying Dart VM doesn't always honor the OS's global proxy configuration. This means even if you have your Burp proxy set up in Android/iOS settings, the traffic just flows straight to the internet.

## The Solution: Interception via Spoofing

By using DNS spoofing, we tell the mobile device that the IP address for `api.target.com` is actually the IP of our local machine running Burp Suite.

### High-Level Architecture

![Flutter Interception Architecture](/assets/images/flutter-intercept-dnsmasq.png)

---

## Step 1: Set Up Burp Suite Invisible Proxying

Since we are redirecting traffic at the IP level, the app will send "unaware" requests to Burp. We need Burp to handle these.

1. Open Burp Suite -> **Proxy** -> **Proxy Settings**.
2. Click **Add** to create a new listener.
3. Bind to **Port 443** (or 80 if the app uses HTTP).
4. Bind to **All interfaces**.
5. Go to the **Request Handling** tab and check **Support invisible proxying (enable only if needed)**.
6. Click **OK**.

> [!IMPORTANT]
> You may need to run Burp as sudo/administrator to bind to port 443.

---

## Step 2: Configure Dnsmasq

`dnsmasq` will act as our rogue DNS server.

1. Install `dnsmasq` (on Linux: `sudo apt install dnsmasq`).
2. Create or edit a configuration file (e.g., `/etc/dnsmasq.conf`).
3. Add the following configuration:

```bash
# Forward other queries to Cloudflare
server=1.1.1.1
# Log queries for debugging
log-queries
log-facility=/var/log/dnsmasq.log

# Spoof the domain to your Local/Proxy IP
address=/api.staging.stavoya.com/192.168.68.109
local=/api.staging.stavoya.com/
```

### Breakdown of the Configuration

- **`server=1.1.1.1`**: Defines the upstream DNS. Requests for non-spoofed domains are forwarded here so the device retains internet access.
- **`log-queries` & `log-facility`**: Logs every DNS request to a file. Useful for discovering hidden endpoints via `tail -f /var/log/dnsmasq.log`.
- **`address=/.../.../`**: The core hijack. Forces the target domain to resolve to your local machine IP.
- **`local=/.../`**: Prevents the query from "leaking" to upstream servers, keeping the interception private.

> [!TIP]
> **What if you don't know the API endpoint?**
> If you aren't sure which domain the Flutter app is calling, simply watch the logs in real-time while using the app:
>
> ```bash
> tail -f /var/log/dnsmasq.log
> ```
>
> Once you spot the target domain in the logs, add a new `address=/domain/IP` entry for it in your config and restart `dnsmasq`.

1. Restart dnsmasq:

   ```bash
   sudo systemctl restart dnsmasq
   ```

---

## Step 3: Configure the Mobile Device

Now, tell your test device to use your machine for DNS.

1. On your Android or iOS device, go to **Wi-Fi Settings**.
2. Change the DNS settings from **Automatic** to **Manual**.
3. Set the DNS server to the IP address of your machine running `dnsmasq`.
4. Ensure the device and your machine are on the same local network.

---

## Step 4: Handle SSL/TLS (The Final Boss)

Even with DNS spoofing, Flutter will notice the SSL certificate doesn't match—unless you've installed Burp's CA certificate.

1. Download the Burp CA certificate.
2. Install it on the mobile device (System trust store or via a tool like Magisk for Android 11+).
3. **Flutter Special Case:** If the app uses **SSL Pinning**, you'll need to bypass it using Objection or Frida.

---

## Conclusion

DNS spoofing with `dnsmasq` is a powerful technique for intercepting "proxy-aware" and "proxy-unaware" applications like those built with Flutter. By hijacking the resolution process, you remove the app's choice to ignore your proxy.

Happy Hacking! 🛡️
