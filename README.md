# Beckhoff IPC Bring-up Notes

## System

- Beckhoff IPC
- Debian GNU/Linux 13 (Trixie)
- PREEMPT_RT kernel
- Default credentials:
  - Username: `Administrator`
  - Password: `1`

  For installation of Beckhoff Linux RT, follow this tutorial:
  https://infosys.beckhoff.com/index.php?content=../content/1031/beckhoff_rt_linux/17350491915.html&id=

## Repository Fix

The Beckhoff repository returned a `401 Unauthorized` error. Fixed by authenticating against the Beckhoff Package Server.

**Authenticate the Beckhoff repository**

```bash
sudo nano /etc/apt/auth.conf.d/bhf.conf
```

```txt
machine deb.beckhoff.com
login user.name@baubot.com
password your_beckhoff_password

machine deb-mirror.beckhoff.com
login user.name@baubot.com
password your_beckhoff_password!
```

Then update:

```bash
sudo apt update
```

## Keyboard Layout

Temporary German layout:

```bash
sudo loadkeys at
```

Permanent change:

```bash
sudo nano /etc/default/keyboard
sudo reboot
```

## SSH Key Setup

Generate an ED25519 SSH key:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

View the public key to copy/verify it:

```bash
cat ~/.ssh/id_ed25519.pub
```

Reference: [GitHub — Generating a new SSH key and adding it to the ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

## Autocompletion

Bash completion isn't installed by default on this image:

```bash
sudo apt update
sudo apt install bash-completion
```

Reconnect via SSH (or `source ~/.bashrc`) for it to take effect.

## Locale Fix (missing `en_US.UTF-8`)

Every new shell warned that `en_US.UTF-8` couldn't be set. This is a minimal Beckhoff image, so the `locales` package was never installed and only `C`, `C.utf8`, and `POSIX` existed (confirmed with `locale -a`), but the login environment already asked for `en_US.UTF-8`, so every shell failed to set it.

To fix this, install the `locales` package and generate the `en_US.UTF-8` locale:
```bash
sudo apt update
sudo apt install locales
sudo nano /etc/locale.gen
# uncomment "# en_US.UTF-8 UTF-8" -> "en_US.UTF-8 UTF-8", save, exit
sudo locale-gen
locale -a
sudo update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
# log out and reconnect (new SSH session), then:
locale
```

## Docker Install

Followed the official tutorial: [Baubot ROS — docker_install.md](https://github.com/Baubot/baubot_ros/blob/jazzy/docker/docker_install.md)

**Deviation from Step 4 of the tutorial:**

The tutorial's command:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

fails with a `404 Not Found`, because this is Debian 13 (Trixie), not Ubuntu — there is no Ubuntu "trixie" release.

**Use this instead:**

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Docker Container Networking (nftables forward chain)

Docker builds that run `apt update`/`apt install` inside a container (e.g. `docker compose ... --build`) failed with `Temporary failure resolving '...'`, even though the host itself had working DNS and internet access.

**Root cause:** Beckhoff RT Linux's own `nftables` ruleset has a restrictive `forward` chain (`table inet filter { chain forward { policy drop; ... } }`) that only allows `established,related` traffic and IPv6 ICMP. Docker installs its own separate NAT/forward rules (`table ip nat`, `table ip filter`) that correctly allow traffic from `docker0`, but nftables evaluates **all** base chains on the `forward` hook — so Beckhoff's chain was silently dropping new (non-established) IPv4 traffic from containers before it ever reached Docker's rules.

**Fix — allow the Docker bridge in Beckhoff's forward chain:**

Add a drop-in file rather than editing Beckhoff's managed config directly:

```bash
sudo nano /etc/nftables.conf.d/docker-forward.conf
```

Contents:

```
add rule inet filter forward iifname "docker0" accept
add rule inet filter forward oifname "docker0" accept
```

**Important — do not use `systemctl restart nftables` to apply this.** It reloads the entire ruleset from disk and wipes out Docker's own dynamically-created NAT/forward rules (they're inserted live by `dockerd`, not stored in a config file), breaking container networking again. Restart Docker instead, which reapplies its rules on top of the current nftables base:

```bash
sudo systemctl restart docker
```

Verify both rule sets are present:

```bash
sudo nft list chain inet filter forward
sudo nft list table ip nat
```

Test container internet access:

```bash
docker run --rm busybox ping -c 3 8.8.8.8
```

**Note for future changes:** any time the nftables ruleset is modified, restart `docker` (not `nftables`) afterward, or restart both — restarting only `nftables` will break Docker networking every time.

## Build Docker Container

```bash
git clone --recurse-submodules git@github.com:Baubot/baubot_ros.git
cd docker
docker compose up -d ros2_manipulation --build
```

## SSH Into the Device

```bash
ssh Administrator@192.168.0.101
```

### Checking / Fixing the Network Interface

When connecting the device to a new network, check the current interface configuration first:

```bash
sudo ip addr show
```

Example output — identify which physical interface is actually `UP` and carries the expected IP:

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
    inet 127.0.0.1/8 scope host lo

2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    link/ether 00:01:05:7b:1c:b5
    inet 192.168.0.101/24 scope global eno1

3: enp2s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 ...
4: enp4s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 ...
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 ...
    inet 172.17.0.1/16 scope global docker0
```

In this case, the active physical interface is `eno1` (the other Ethernet ports, `enp2s0`/`enp4s0`, show `NO-CARRIER` — nothing plugged in).

If `eno1` doesn't have the expected IP (e.g. `192.168.0.101/24`), set it manually:

```bash
sudo ip addr flush dev eno1
sudo ip addr add 192.168.0.101/24 dev eno1
```

**Note: this is not persistent.** `ip addr add` only changes the live, in-memory kernel state, it writes nothing to disk. After a reboot, the interface reverts to whatever config actually exists on disk (or DHCP, if nothing claims the interface), and the manually-added IP is gone.

### Making the Static IP Persistent (systemd-networkd)

This system is managed by `systemd-networkd` (confirm with `systemctl status systemd-networkd`; there is no NetworkManager on this image).

Check for an existing config first:

```bash
ls -la /etc/systemd/network/
cat /etc/systemd/network/*.network
```

Create/edit the interface config:

```bash
sudo nano /etc/systemd/network/10-eno1.network
```

```ini
[Match]
Name=eno1

[Network]
Address=192.168.0.101/24
Gateway=192.168.0.1
DNS=8.8.8.8
DNS=1.1.1.1
```

Apply and verify:

```bash
sudo systemctl restart systemd-networkd
ip addr show eno1
```

Confirm it actually survives a reboot:

```bash
sudo reboot
```
```bash
ip addr show eno1
```

Reference: [Linux `ip` command cheat sheet](https://www.thomas-krenn.com/en/wiki/Linux_ip_command)

## ROS 2 Cross-Machine Networking (PC ↔ Beckhoff)

Getting `ros2 topic pub`/`echo` working between the PC and the Beckhoff involved several distinct issues, layered on top of each other. Documented in the order they were actually hit.

### 1. Confirm both machines' actual current IPs

```bash
ip addr show
```

Both the PC and the Beckhoff run ROS 2 nodes inside Docker containers. ROS 2's DDS discovery relies on UDP multicast and on each participant announcing its own reachable IP as a "locator." Under Docker's default bridge networking, that's the container's internal IP (e.g. `172.17.0.x`), which the other machine can never reach, so DDS discovery and data both silently fail even if the network path itself is fine.

**Fix:** use host networking for any container running ROS 2 nodes across machines. In `docker-compose.yaml`:

```yaml
services:
  your_ros2_service:
    network_mode: host
```

### 3. Beckhoff's nftables input chain blocks DDS ports by default

Even with host networking and the correct IPs confirmed, `ros2 topic` communication into the Beckhoff didn't work, because the Beckhoff's nftables `input` chain only allowlists specific ports (`22, 443, 8016, 48899, 5353`) — the same restrictive-by-default pattern seen earlier with the Docker `forward` chain. CycloneDDS's discovery and data traffic (UDP, multicast group `239.255.0.1`, port `7400` and up) wasn't in that list.

```bash
sudo nano /etc/nftables.conf.d/ros2-dds.conf
```

Contents:

```conf
add rule inet filter input udp dport 7400 ip daddr 239.255.0.1 accept
add rule inet filter input udp dport 7400-7500 accept
add rule inet filter input udp dport 49150 ip daddr 225.0.0.1 accept        # for ros2 multicast send/receive, see note below
```

**Note on the `ros2 multicast` CLI tool:** it's used to sanity-check multicast connectivity, but it uses its own hardcoded address, **`225.0.0.1:49150`**, completely separate from DDS's actual discovery address (`239.255.0.1:7400`). A failure in `ros2 multicast receive` does **not** mean real ROS 2 topics won't work, and a pass doesn't guarantee they will. The third rule above is only needed if you want that specific test tool to work; it's not required for actual ROS 2 topics.

Apply, remembering the Docker/nftables gotcha from before (restarting `nftables` wipes Docker's live rules — restart both):

```bash
sudo systemctl restart nftables
sudo systemctl restart docker
```

### Verify with real topics, not the multicast tool

```bash
# On the Beckhoff:
ros2 topic pub /chatter std_msgs/String "{data: 'from Beckhoff'}"

# On the PC:
ros2 topic echo /chatter
```

### Result

With both machines' IPs confirmed (and made persistent), `network_mode: host` on both containers, and the nftables rules above, `ros2 topic pub`/`echo` works in both directions between the PC and the Beckhoff.

## Installing TwinCAT Runtime

Reference: https://infosys.beckhoff.com/index.php?content=../content/1031/beckhoff_rt_linux/17350491915.html&id=

```bash
sudo apt install tc31-xar-um
```

Verify:

```bash
sudo systemctl status TcSystemServiceUm
```

## ADS Layer
### CLI ADS Tool

1. SSH into Beckhoff

```bash
ssh Administrator@192.168.x.x
```

Use this for printing the NetId 
```bash
tcadstool 127.0.0.1 netid
```

- Most useful commands:
Print examples
```bash
tcadstool --help
```
Show ADS Variables:
```bash 
tcadstool 5.123.28.181.1.1 plc show-symbols
```

Read ADS Variable:
```bash
tcadstool 5.123.28.181.1.1 plc read-symbol "MAIN.bRunOnlyOnce"
```

Write ADS Variable:
```bash
tcadstool 5.123.28.181.1.1 plc read-symbol "MAIN.bRunOnlyOnce"
```
