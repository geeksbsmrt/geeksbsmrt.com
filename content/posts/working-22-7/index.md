---
title: "Working 22/7"
summary: "Career, family, a home lab, and a blog, with no dessert in sight."
date: "2025-05-27"
series: ["RaspberryPi"]
series_order: 1
---

Working as a full-time Client Platform Engineer, raising a family, setting up a home lab, and blogging has me feeling like I get no sleep. I feel like I'm working 22 hours a day, 7 days a week. I'm not, but my Raspberry Pi is.

## Where It Started

When my kids started using their own electronic devices on the internet, I needed a way to ensure they didn't find their way onto inappropriate websites. There are many apps and services that you can subscribe to which will do this for you. Being an IT guy, and having a background in mobile devices, I figured I could come up with something just as good without the subscription.

After time exploring and comparing options, I settled on PiHole. There were similar options like AdGuard, OpenDNS, and NextDNS. However, I wanted more control and flexibility in my selection. PiHole could be self-hosted and gave full control over DNS block lists, hardware groups, and even provides a decent DHCP server.

### The Hardware

I picked up a [CanaKit Raspberry Pi 4 8GB Extreme Kit](https://www.canakit.com/raspberry-pi-4-extreme-aluminum-case-kit.html?srsltid=AfmBOorioMuCoapVp045hg8nDOhgeewRR1Og87KC5_Z5R83ht22GjbL1). This kit came with the Raspberry Pi, power cord and switch, MicroSD card, and other accessories to get you started. These are great little devices with a number of uses. People were making smart mirrors, digital calendars, greenhouse sensors, and more. I had a friend who made a sentry turret that shot foam darts at you when you came within range.

### The Operating System

The Raspberry Pi can also run a full install of many Linux distributions. [Raspberry Pi OS](https://www.raspberrypi.com/software/operating-systems/), previously Raspbian, was a popular choice. It is maintained by the Raspberry Pi Foundation to be the most compatible OS with their hardware. However, at the time Raspbian Buster came in 2 options: 32-bit or [beta](https://www.raspberrypi.com/news/raspberry-pi-os-64-bit/) 64-bit. I was looking for a stable ARM64 OS, so I decided to run [Ubuntu Server 20.04](https://cdimage.ubuntu.com/ubuntu/releases/20.04/release/). Ubuntu had been a solid server OS for decades, leading to a large community for support when needed.

I needed a way to get my new OS onto my microSD card. Luckily Raspberry Pi had planned for this and built the [Raspberry Pi Imager](https://www.raspberrypi.com/software/). This utility allows you to image, or load an operating system, onto a drive. You may be familiar with similar applications, such as [Rufus](https://rufus.ie/en/).

### The Software

I knew enough to ensure I started with a fully up-to-date system. This keeps the system secure and stable. I had used some Linux here and there around my IT support positions, but I still would not call myself experienced. After updates, it was time to install [PiHole](https://docs.pi-hole.net/main/). I followed the manual installation instructions provided in their documentation and was up and running with websites being blocked correctly.

**_NOTE: The docs provide an automated installer. It requires you to pipe `|` to bash. While convenient, I urge caution running any script like this. Make sure you trust the author, source, and content of the script before running. Security risks involving DNS Spoofing, Man-in-the-Middle attacks, source takeover, and more are still possible._**

Talking about DNS Spoofing, what would happen if my PiHole's upstream DNS got hijacked? My entire network would be affected. To overcome this highly unlikely scenario, and just because I could, I installed [Unbound](https://www.nlnetlabs.nl/projects/unbound/about/) as an upstream DNS following the guide in Pi-Hole's [documentation](https://docs.pi-hole.net/guides/dns/unbound/).

### The Configuration

The [default](https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts) block list shipped with Pi-Hole is a great starting point. However, it's primary purpose is to block ads and known malicious sites. I applied this list as default to all devices. Next, I looked for block lists that would filter those 'inappropriate' sites I wanted to restrict my children's devices from getting to. I ended up with a [list](https://github.com/hagezi/dns-blocklists/) blocking adult sites, gambling, file sharing, and social media.

This is where I ran into trouble. Not so much with the Pi-Hole, but with my wife who quickly informed me that I had broken the internet. Oops? I disabled the newly added list to resolve her issue, then set about configuring device groups. These groups allow assigning specific lists to sets of devices. I created groups for my IoT devices, my two youngest children, my teenager, and my wife and myself. I applied the more restrictive list to my youngest's group and re-enabled it. My wife was able to continue using her Facebook, and I no longer had to sleep on the couch. For my teen, I used a similar list to the youngest's but allowed social media.

For a few years, the Pi had served me well. Over that time, I kept tinkering with its configuration and the software installed on it. For instance, I have a second domain that I purchased when I was in college. I figured I could dust off my web development skills and put together a landing page. I knew of IIS for Windows, and the last time I worked with a Linux server, Apache was your web server; Nginx was just starting. I wondered what the options were 15 years later. That's when I found [Caddy](https://caddyserver.com/), a lightweight Go-based, webserver that automatically manages your TLS certificates.

I also switched Pi-Hole's admin UI from Lighttpd to Caddy so I only had one webserver running on the Raspberry Pi. This process took more time and caused more frustration than I'd often like to admit. At the time, Pi-Hole's admin UI used PHP and I ran into issues configuring `php-fastcgi` support correctly in Caddy. After digging through the countless Google search results (all these fancy AI tools weren't available) most referencing [Caddy Community](https://caddy.community/), [Pi-Hole's Discourse](https://discourse.pi-hole.net/), and [StackOverflow](https://stackoverflow.com/questions), I was up and running. Ok, maybe more like not breathing and backing away cautiously, averting my gaze. If it's not broke, don't acknowledge it. Right?

## Where It's At

But it's a home lab, it's supposed to be poked, prodded, and punished. Now here I am, years later, writing a blog post on how exactly I'm breaking a working system.

### The Why

Me. I can't help but tinker. I love trying new technologies and learning new things. But if I tinker with something the wrong way and can't easily get back, forget the couch, I'll be sleeping in the car! So what can I do to ensure that I can quickly and easily get back to a working configuration in the event it goes sideways?

### The What

[Docker](https://www.docker.com/). Containers are small, [Docker Compose](https://docs.docker.com/compose/) makes configuration simple, [Git](https://git-scm.com/) can track changes and rollback quickly, [GitHub](https://github.com/) is a great place to back-up (and [show off](https://github.com/geeksbsmrt)), and GitHub actions can validate and deploy my changes. With the enhanced workload I'm beginning to throw at the Raspberry Pi, the MicroSD card couldn't keep up. I purchased a USB 3.0 SSD and loaded the now full production Raspberry Pi OS Lite 64-bit.

### The How

[Gemini](https://gemini.google.com/). I've given in to our technoverlords. Also, if I don't keep current and use the tools available to me, I'll become obsolete. If I'm learning, I'm gonna soak up the sum. C'mon, you knew there were going to be more puns. That one's a toof-er, the things my youngest tries to pull out of her mouth when they're loose.

## Where It's Going

Check back soon for the next part of the Raspberry Pi series. I'll be detailing my Docker Compose file and how my home infrastructure is configured. Sanitized, of course :wink:
