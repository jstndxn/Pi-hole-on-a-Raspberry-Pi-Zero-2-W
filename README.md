# Pi-hole on a Raspberry Pi Zero 2 W

Network-wide DNS ad blocking on a $15 computer, set up headless over SSH.

This is a working record of an actual build, including the parts that went wrong. Most guides assume a clean run. This one didn't have one, and the failures turned out to be the useful part.

**Result:** 81,395 blocked domains, ~2W power draw, running 24/7.

---

## Hardware

| Item | Notes |
|---|---|
| Raspberry Pi Zero 2 W | 512MB RAM, 2.4GHz Wi-Fi only |
| microSD card, 16–32GB | Use a decent brand. Cheap cards are the main long-term failure mode |
| 5V 2.5A micro-USB supply | Underpowered supplies cause Wi-Fi dropouts |
| Ethernet HAT *(optional)* | Recommended — Wi-Fi is the weak link for a DNS server |

**Gotcha:** the Zero has two micro-USB ports. The **outer** one (labelled PWR IN) is power. The inner one is data. Power in the wrong port does nothing at all.

---

## Setup

### 1. Flash the card

Raspberry Pi Imager → Device: **Pi Zero 2 W** → OS: **Raspberry Pi OS Lite (64-bit)**.

At the "apply OS customisation settings?" prompt, click **Edit Settings**. Do not skip this — it's the only chance to configure Wi-Fi, and without Wi-Fi there's no way to reach the Pi.

- Hostname: `pihole`
- Username + password (write these down)
- Wi-Fi SSID, password, and **wireless LAN country** — the radio stays off without the country set
- **Services tab → Enable SSH → password authentication**

The Services tab is a separate tab from the Wi-Fi settings. Missing it is the single most common setup failure. See troubleshooting below.

### 2. Boot

Card in, power into the outer port, wait 3–5 minutes. No screen, no feedback. First boot is slow.

### 3. Connect

```bash
ssh <username>@pihole.local
```

### 4. Update

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

20–40 minutes on a Zero 2 W. Don't interrupt it.

### 5. Reserve the IP

The Pi's address must never change or the whole network loses DNS. Set a DHCP reservation in the router, bound to the Pi's MAC:

```bash
ip addr show wlan0   # the link/ether line is the MAC
```

### 6. Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

| Prompt | Answer |
|---|---|
| Interface | `wlan0` |
| Upstream DNS | Cloudflare (DNSSEC) |
| StevenBlack's blocklist | Yes |
| Web admin + web server | Yes |
| Query logging | Yes |
| Privacy level | 0 — show everything |

### 7. Post-install

```bash
sudo pihole setpassword
sudo usermod -aG pihole $USER    # log out and back in to take effect
```

### 8. Verify

```bash
dig doubleclick.net @127.0.0.1 +short   # expect 0.0.0.0
dig github.com @127.0.0.1 +short        # expect real IPs
```

A sinkholed ad domain and a working lookup. Both must be true.

### 9. Point clients at it

Set DNS to the Pi's IP. **Leave the secondary blank** — a fallback like `8.8.8.8` means devices silently bypass Pi-hole at random.

---

## Troubleshooting log

### `Connection refused` on port 22

Refused is different from timed out. Refused means the hostname resolved and something answered — the Pi is on the network, SSH just isn't running. Almost always the Services tab was missed during flashing.

Fix without reflashing: put the card in a computer and create an empty file named `ssh` (no extension) on the `bootfs` partition.

```bash
touch /Volumes/bootfs/ssh
```

On Windows, enable **View → File name extensions** first, or you'll silently create `ssh.txt`, which does nothing.

### SSH session freezes mid-command

Keystrokes going into a dead socket after the link dropped. Escape with `Enter` `~` `.`

Usually an underpowered supply or weak Wi-Fi — the Zero browns out and the radio is the first thing to go. Connect with a keepalive so a dead link reports itself instead of hanging:

```bash
ssh -o ServerAliveInterval=30 <username>@<pi-ip>
```

### Password prompt appears to do nothing

It's working. SSH shows no characters, no dots, no cursor movement while typing a password. Type it and press Enter.

### Wrong subnet

Guides use `192.168.1.x` as the example range. Plenty of networks (Comcast included) use `10.0.0.x`. Pinging a guessed address returns 100% loss regardless of whether the Pi is fine.

```bash
route -n get default | grep gateway   # macOS
ipconfig                              # Windows — Default Gateway
```

In macOS System Settings → Network → Wi-Fi → Details → TCP/IP, **IPv4 Address** is your own machine. **Router** is the gateway. Easy to confuse.

### Router admin page unreachable despite the gateway responding to ping

Check for VPN tunnels (`utun0`, `utun1`…) in `netstat -rn`, and for iCloud Private Relay. Either will route traffic off the local network and produce `ERR_ADDRESS_UNREACHABLE` on a gateway that's otherwise perfectly healthy.

### `Failed to set dns.upstreams. Try with sudo power`

**The important one.** The installer finished but never wrote the upstream resolvers — Pi-hole would block ads and then fail to resolve anything. It also exits without showing the completion screen or the generated password.

```bash
sudo pihole-FTL --config dns.upstreams '[ "1.1.1.1", "1.0.0.1" ]'
sudo pihole-FTL --config dns.upstreams    # verify it echoes back
sudo systemctl restart pihole-FTL
```

Then run Gravity manually, since it likely got skipped too:

```bash
sudo pihole -g
```

Expect a domain count around 81,000 for StevenBlack's list. A count of 0 means the download failed.

### Ad domains still resolving after install

Gravity never ran, or the old answer is cached:

```bash
sudo pihole -g
sudo pihole restartdns
pihole query doubleclick.net   # shows which lists contain a domain
```

### Non-errors that look like errors

- `SELinux not detected` — informational. Raspberry Pi OS doesn't use it
- `[✗] Check for existing repository` — no prior install to find. Correct on a fresh setup
- `Neutrino emissions detected...` — the developers being silly

---

## Xfinity / Comcast limitation

**xFi Gateways (XB6, XB7, XB8) lock the DNS field.** It's hard-coded into the firmware and the mobile app doesn't expose it at all. Step 9 — the step that makes this network-wide — is unavailable on a leased Comcast gateway.

Options:

1. **Per-device DNS** — free, works immediately, doesn't cover guests or devices with no DNS setting
2. **Bridge mode + your own router** — full control. Disables xFi Pods and xFi Advanced Security
3. **Buy your own modem/router** — also ends the rental fee

Test with option 1 before spending anything. Note that some firmware reportedly intercepts port 53 and redirects to Comcast's resolvers even when DNS is set per-device — confirm queries actually appear in the dashboard.

---

## Client setup

**macOS** — System Settings → Network → Wi-Fi → Details → DNS

Note that on older macOS this is a global interface setting, not per-network. Taking the laptop to another network leaves it pointed at an address that doesn't exist, and nothing resolves. On Ventura+ it's stored per-network.

**iOS** — Settings → Wi-Fi → ⓘ → Configure DNS → Manual. Stored per-network, so it applies at home and ignores itself elsewhere. Better place for this than a laptop.

**LG webOS** — All Settings → General → Network → Wi-Fi Connection → Advanced Wi-Fi Settings → Edit → untick Set Automatically. Requires a full static config: IP, subnet `255.255.255.0`, gateway, DNS.

Worth doing alongside: General → System → Additional Settings → disable **Live Plus** (automatic content recognition, fingerprints everything on screen including HDMI inputs) and **Home Promotion**.

**Roku** exposes no DNS setting at all. Router-level only.

---

## What it does and doesn't block

Pi-hole filters by **domain name**. If an ad comes from a different domain than the content, it dies. If it comes from the same domain, blocking it would break the site.

**Blocked:**
- Display ads on websites
- Ads inside free mobile apps and games
- Tracking, analytics, ad-tech beacons — the bulk by volume
- Device telemetry, especially smart TVs
- Malware/phishing domains with the right lists

**Not blocked:**
- Any streaming service — Peacock, Paramount+, Disney+, Hulu, Max, Netflix, Prime Video. All use **server-side ad insertion**: ads are stitched into the video stream from the same servers as the content. There is no separate domain to block
- YouTube ads, for the same reason
- Sponsored posts in social feeds — those *are* the content
- Podcast ads baked into the audio
- Anything over a VPN or using DNS-over-HTTPS

Expect 15–30% of queries blocked. The noticeable wins are the open web, free apps, and a large drop in background tracking. It is not a streaming ad blocker. Nothing is.

For YouTube specifically: Firefox + uBlock Origin. Full uBO no longer runs on stable Chrome following the Manifest V3 transition.

---

## Maintenance

```bash
sudo apt update && sudo apt full-upgrade -y
pihole -up
```

Monthly. Pi-hole is a rolling release and security fixes land regularly.

**When something breaks**, Pi-hole is suspect number one. Find the domain in the dashboard's Query Log, then:

```bash
pihole allow example.com
pihole disable 5m     # escape hatch while diagnosing
```

**Back up** via Settings → Teleporter → export, and keep it off the Pi. Saves repeating all of this when the SD card eventually fails.

**Keep it single-purpose.** At 512MB, adding Docker or a media server is how you meet the OOM killer.

---

## Next

- **WireGuard** (via PiVPN) — runs fine alongside Pi-hole on the same Zero. Gives ad blocking on cellular and remote access to the home network. Needs UDP port 51820 forwarded and a dynamic DNS name
- **Tailscale** — same outcome, no port forwarding or dynamic DNS. The answer if you're behind CGNAT

---

## License

MIT
