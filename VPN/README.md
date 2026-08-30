My very first home lab application - WireGuard VPN + AdGuard adblock

# Design
The idea is that the host machine acts as a VPN server that makes internet requests on behalf of the client. Each request undergoes a DNS filter to filter out any DNS requests that "look like ads". As such, the goal is that no ads make it through to the client.

## WireGuard
The VPN runs on a WireGuard Docker container. It listens for udp traffic on this machine's port `51820`, forwards DNS queries to AdGuard, and sends the responses back to the client.

WireGuard's admin UI is available on this machine's port `51821`. However this is only available on the LAN and not exposed to the internet.

## AdGuard
We have configured `172.20.0.2` to be the static IP address of the AdGuard DNS server (*). Devices connected to the WireGuard VPN forward their DNS queries to `172.20.0.2`, which then filters out "ad-like" DNS queries, and only returns the genuine ones. See `Notes -> What is DNS` for more.

AdGuard exposes an admin UI on port `3000`. Like WireGuard, this is only available on the LAN and not exposed to the entire internet.

(*): Not really "static IP address of DNS server", in reality it is the static IP address of AdGuard DNS server *within* the Docker network `vpn_net`, which is assigned explicitly in `docker-compose.yml`. However for our purposes, we can just imagine it as a static IP address of a DNS server.

# Setting Up

## Server Side
- Apply router port-forward rule: UDP `51820` → `<LAN IP of this machine>:51820`. For WireGuard connections
- Apply router port-forward rule: TCP `22` → `<LAN IP of this machine>:22`. For ssh connections
- Edit the DNS records for `kerdi.ca`. This should be done already, but if not add a DNS record
    - `Type: A`
    - `Host: vpn.kerdi.ca`
    - `Answer / Value: <Lookup the IPv4 address of this machine>`
    - `TTL: 600`
- Install `docker compose`. Run `sudo apt update && sudo apt install docker-compose-v2`, verify with `docker compose version`. Should see something like `Docker Compose version v2.40.3...`
- Enable and persist `net.ipv4.ip_forward=1`. Simply open the file `/etc/sysctl.d/99-homelab-wireguard-ipv4forward.conf` and write the line `net.ipv4.ip_forward=1`.

Verify with `sysctl net.ipv4.ip_forward`. Should see `net.ipv4.ip_forward = 1`

This configuration option permits the kernel to forward packets between network interfaces, allowing us to forward VPN client traffic to the Internet.

- Open the file `/etc/modules-load.d/99-homelab-wireguard-iptablesmodules.conf` and write the following lines
```
ip_tables
iptable_nat
ip6_tables
ip6table_nat
```
WireGuard clients will receive addresses like `10.8.0.2`. These are private addresses that are only meaningful within the WireGuard network. A device on the public internet does not know how to reach `10.8.0.2`. That is like addressing a letter to "Apartment 306". If a WireGuard client sends a packet to somewhere with source address `10.8.0.2`, the reply would have nowhere valid to come back to.

NAT: Network Address Translation. Rewrites the address so it looks like it came from a real public internet address.

Configuring these options allows WireGuard to access the tools to perform this.

- Configure `ufw`: rate-limited SSH (`22/tcp`, v4+v6), allowed WireGuard (`51820/udp`, v4+v6), default `deny incoming` / `allow outgoing`

    1. `sudo ufw limit OpenSSH` - allow ssh connections to this machine (on port 22). `limit` instead of `allow` to guard against brute-force - deny an IP after repeated failed connection attmepts in a short window.
    2. `sudo ufw allow 51820/udp` - allow traffic to the WireGuard UDP port.
    3. `sudo ufw default deny incoming` and `sudo ufw default allow outgoing` - allow this machine to send any outbound packets. Deny any packets that have not been explicitly allowed by the above rules
    4. `sudo ufw enable`
    5. `sudo ufw status verbose` should look like this
    ```
    Status: active
    Logging: on (low)
    Default: deny (incoming), allow (outgoing), deny (routed)
    New profiles: skip

    To                                                 Action            From
    --                                                 ------            ----
    22/tcp (OpenSSH)                     LIMIT IN        Anywhere
    51820/udp                                    ALLOW IN        Anywhere
    22/tcp (OpenSSH (v6))            LIMIT IN        Anywhere (v6)
    51820/udp (v6)                         ALLOW IN        Anywhere (v6)
    ```
- Confirm `unattended-upgrades` is properly configured and actively running. This ensures ubuntu (or whatever OS runs this lab) updates itself without requiring human input. Important because this machine has UDP and ssh ports exposed to the internet, so unpatched vulnerabilities in kernel, SSH, etc. would be risky. This option downloads and installs these patches automatically.

For updates that require a reboot, the system will *not* reboot on its own. This will require manual input.

- Set a DHCP reservation for this machine's LAN IP on the router so it never changes. Ensure this machine always gets the same IP address on this LAN so the port forwarding rule is not broken.

Go to the router admin UI and find this machine under "connected devices". Change its mode from `DHCP` to `Reserved IP`.

To verify, force a full DHCP release and renew, confirm the router gives the same address.
1. `ip addr show enp2s0` - remember the `inet` address
2. `sudo nmcli connection down netplan-enp2s0 && sudo nmcli connection up netplan-enp2s0`
3. `ip addr show enp2s0` - should show the same `inet` address as step 1

At the time of writing, this machine uses NetworkManager (`nmcli`). Others may use `dhclient`. If this is the case, ask your favorite LLM how to perform a similar check.

- `cd lab/VPN` + `docker compose up -d` - start the WireGuard and AdGuard docker containers as defined in `docker-compose.yml`
- Complete AdGuard admin UI setup. On a broswer, go to `<LAN_IP>:3000`. Setup according to the instructions. Choose any sensible Upstream DNS server, choose your own block lists, etc
    - Admin Web Interface - the GUI for AdGuard home. Make this listen on all interfaces on port 3000.
    - DNS Server - the listener for DNS requests. Make this listen on `172.20.0.2` port `53`.
        - Earlier, we defined the address `172.20.0.2` to be the address of the DNS server. Port `53` is default.
- Complete WireGuard admin UI setup. On a broswer, go to `<LAN_IP>:51821`. Enter `Host = vpn.kerdi.ca` and `Port = 51820`.
- Confirm the admin UIs are not reachable from outside the general internet. Try hitting `http://<IP address>:51821` and `http://<IP address>:3000` from a phone on cellular data, should not work.

## Adding a Client
- On the WireGuard admin UI panel, add a client. Configure the client's DNS field to be `172.20.0.2` - the IP address we assigned already to the AdGuard DNS server.

- On the cilent side, get the free WireGuard app. Add the client by scanning the QR code. Once the connection is successful, browse as usual!

# TODO
Write a script to automatically update Porkbun's DNS records to keep `vpn.kerdi.ca` pointing to this machine. In theory, my public IP address can change at any time and we need `vpn.kerdi.ca` to resolve to the correct address.

This is not a priority since I have not noticed my IP address change in over a year.

# Known Limitations
This pipeline performs Ad-Block on a DNS level. Some applications like YouTube run their ads from the same domain as the videos; as such, this DNS based ad-block will not work.

# Notes
1. How does Docker know to automatically start up these services on bootup?

In `docker-compose.yml`, we specify that both wireguard and adguard should start automatically on startup. How does it know this?

`docker-compose.yml` is only used when I run `docker compose up`. This reads the file, creates the containers, and saves the policy into each container's persistent config. After this, that file is not used anymore.

On startup, the docker daemon starts, and it launches every container it knows about, including our 2 of interest, and starts them automatically (because they have been configured to do so).

2. What is DNS
Internet devices are identified by an IP address. Domain Name System (DNS) translates human-readable names into IP addresses.
Public DNS servers like `8.8.8.8` perform the lookup, and return the human-readable name to IP address translation.

In my pipeline, we forward DNS queries to AdGuard. We maintain a list of domains that are known to serve ads. If we request a domain that is on a blocklist, AdGuard will just answer `0.0.0.0` or refuse to perform the lookup at all. For anything else, AdGuard will make a query to a public DNS server like usual. The result is no "ad-like" traffic reaches the client device.