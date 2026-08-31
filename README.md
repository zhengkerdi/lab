This repository contains everything you need to reproduce my home lab on a fresh machine. The purpose of this project is to gain experience with networking, security, and open source software. Although AI was used to help in research and debugging, everything pushed is written by me.

# Hardware
Originally, I tried to source the hardware to host the lab by dumpster diving at the Burnaby Recycle Center. 5 minutes into my treasure hunt, some chud worker kicked me out because 🤓☝️"those are City of Burnaby property". I did manage to abscond with 2 sticks of 1333 MHz DDR3 memory totaling 6 GB (4GB + 2GB, street value $4 CAD). Later attempts at plundering the same recycle center were met by the same worker diligently watching over his pile of garbage.

So, I acquiesce and obtain my machine from Facebook Marketplace. The machine is a Lenovo ThinkCenter workstation from 2013, it came with an Intel Core i5-4570 CPU, 8GB of 1333 MHz DDR3 RAM, and a 1TB hard drive. The posting was originally for $35, but after some haggling I bought it for $30. Remember those 2 sticks of RAM from the recycle center? The motherboard happens to have 2 free slots, so I insert my 2 garbage sticks and now the machine has a total of 14 GB of 1333 MHz DDR3 memory.

On my new supercomputer, I install Ubuntu Server 26.04.1 and now we are ready to build!

# Applications
## VPN + DNS AdBlock
On this machine, we host a WireGuard VPN and route all its clients’ traffic to an AdGuard DNS server that blocks any DNS requests to "ad-like" domains. In addition to AdBlock, this also gives us the ability to VPN into the LAN and access things such as Admin UI panels from on the go. See `VPN/README.md` for more.

## Jellyfin
Netflix for free! Assuming we have mp4 files (sourced legally of course) stored on the lab machine hard drive, Jellyfin allows us to host a Netflix-like interface to stream these videos from any client. In addition to a web app, Jellyfin has a native app for iOS and many smart TVs, so the UI is excellent. See `Jellyfin/README.md` for more.
