# Self-Hosted Minecraft Java + Bedrock + Console Crossplay (No Port Forwarding)

A single self-hosted Minecraft **Java (Paper)** server that players can join from **Java Edition** and from **Bedrock Edition** on PC, mobile, Nintendo Switch, Xbox, and PlayStation — **without forwarding any ports** on your router.

Crossplay is handled by [Geyser](https://geysermc.org) + [Floodgate](https://wiki.geysermc.org/floodgate/). Remote access is handled by a tunnel (for Java) and by Xbox Live broadcasting (for Bedrock and consoles).

Built and tested on Arch Linux running [Crafty Controller](https://craftycontrol.com) in Docker, but the pieces apply to any Linux host.

## Contents

- [Why it is built this way](#why-it-is-built-this-way)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Part 1 — Base server](#part-1--base-server)
- [Part 2 — Geyser configuration](#part-2--geyser-configuration)
- [Part 3 — Docker host networking](#part-3--docker-host-networking)
- [Part 4 — Host-level fixes](#part-4--host-level-fixes)
- [Part 5 — Java access via playit.gg](#part-5--java-access-via-playitgg)
- [Part 6 — Bedrock and console access via MCXboxBroadcast](#part-6--bedrock-and-console-access-via-mcxboxbroadcast)
- [Part 7 — Server tweaks](#part-7--server-tweaks)
- [How each player joins](#how-each-player-joins)
- [Troubleshooting](#troubleshooting)
- [Making it survive a reboot](#making-it-survive-a-reboot)

## Why it is built this way

Three obstacles shape everything below:

1. **No port forwarding.** You cannot open ports on the router, so the server is not directly reachable from the internet. Java uses a tunnel; Bedrock and consoles use Xbox Live as the path in.
2. **NetherNet.** Since Bedrock 26.60 (Oct 27 2026), Bedrock clients connect only over NetherNet (a WebRTC transport). It needs a separate "signaling" step and does not survive a plain forward-only UDP tunnel. We use the free Warden external signaling service plus MCXboxBroadcast.
3. **Consoles.** PlayStation and Xbox have no "Add Server" button and block direct IP entry. MCXboxBroadcast makes the server appear as a joinable session in the in-game **Friends** tab.

## Architecture

| Piece | Role |
|---|---|
| Paper | The Java server (port 25565) |
| Geyser | Bridges Bedrock clients to the Java server (port 19132, NetherNet) |
| Floodgate | Lets Bedrock players join without owning Java Edition |
| Warden (NXS) | Free external signaling so Bedrock clients can find the server |
| playit.gg | Tunnel giving Java players a public address without port forwarding |
| MCXboxBroadcast | Geyser extension that broadcasts the server over Xbox Live so Bedrock and console players join from the Friends tab |

## Prerequisites

- A Linux host with **Docker** and **Docker Compose**.
- **Crafty Controller 4** (Docker image `registry.gitlab.com/crafty-controller/crafty-4:latest`).
- A **playit.gg** account (free).
- One **throwaway Microsoft / Xbox account** to act as the Bedrock "friend bot" (do not use your main account).
- The latest **Geyser-Spigot** and **Floodgate** plugin jars.
- `sudo` on the host.

Throughout, replace `~/crafty` with your Crafty compose folder and `<SERVER-UUID>` with your server's folder name (shown in Crafty, or as the directory under `docker/servers/`).

## Part 1 — Base server

1. Stand up Crafty Controller via Docker Compose and create a **Paper** server.
2. Put `Geyser-Spigot.jar` and `floodgate-spigot.jar` in the server's `plugins/` folder.
3. Start the server once so Geyser generates `plugins/Geyser-Spigot/config.yml`, then stop it.

Find the admin login Crafty generates in `~/crafty/docker/config/default-creds.txt` (user `admin`). Change the password after first login.

## Part 2 — Geyser configuration

Edit `plugins/Geyser-Spigot/config.yml`. The settings that matter for this setup (see [`examples/geyser-config.yml`](examples/geyser-config.yml)):

```yaml
bedrock:
  address: 0.0.0.0
  port: 19132
  webrtc-port: 0          # 0 = use the same port (19132) for NetherNet media
  transport: nethernet
  clone-remote-port: false
  signaling:
    mode: hybrid          # builtin (local) + nxs (Warden), see note
    advertise-addresses: []
    nxs:
      token: ''
      endpoint: https://agent.warden.cloud

advanced:
  nxs:
    assisted-joins: true
```

**Why `mode: hybrid`.** `nxs` alone registers with Warden but runs no local signaling listener. MCXboxBroadcast hands console players off to Geyser at `127.0.0.1:19132` and needs something listening there, which only `builtin` or `hybrid` provides. `hybrid` keeps Warden working **and** gives MCXboxBroadcast its local endpoint. If you do not need console support you can use `nxs`.

Use Floodgate auth so Bedrock players join without a Java account (`auth-type: floodgate`).

## Part 3 — Docker host networking

Docker's default bridge network sits a second layer of NAT in front of Geyser, which breaks NetherNet's WebRTC return path. Put the Crafty container on the host network instead, and force the JVM to IPv4 (see Part 4). Full file: [`examples/compose.yml`](examples/compose.yml).

```yaml
services:
  crafty:
    container_name: crafty_container
    image: registry.gitlab.com/crafty-controller/crafty-4:latest
    restart: always
    network_mode: host
    environment:
      - TZ=America/New_York
      - JAVA_TOOL_OPTIONS=-Djava.net.preferIPv4Stack=true
    volumes:
      - ./docker/backups:/crafty/backups
      - ./docker/logs:/crafty/logs
      - ./docker/servers:/crafty/servers
      - ./docker/config:/crafty/app/config
      - ./docker/import:/crafty/import
```

> With `network_mode: host` you **remove** the `ports:` block — host networking binds ports directly and conflicts with port mappings.

Recreate the container after editing:

```bash
cd ~/crafty && sudo docker compose down && sudo docker compose up -d
```

Volumes are bind mounts, so `down` never touches your worlds.

## Part 4 — Host-level fixes

These three bit us and are easy to miss.

**Firewall.** Once on host networking, the host firewall is in the path (Docker's published-port rules used to bypass it). Open the Bedrock port:

```bash
sudo ufw allow 19132/udp
```

**Clock.** Warden rejects requests signed too far from its own time. Keep the host clock synced:

```bash
sudo timedatectl set-ntp true
timedatectl          # expect: System clock synchronized: yes
```

**IPv6 (if broken).** If MCXboxBroadcast logs `HTTP connect timed out` reaching Xbox Live, or you see `Send failed, errno=101` and STUN cannot reach IPv6 addresses, your host has broken IPv6 routing. Java tries Xbox Live over IPv6 first and hangs. Disable IPv6 on the host (the host-networked container inherits this):

```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1 net.ipv6.conf.default.disable_ipv6=1
```

Make it permanent in `/etc/sysctl.d/99-disable-ipv6.conf`:

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

Skip this if your IPv6 works. Undo with the same keys set to `0`.

## Part 5 — Java access via playit.gg

1. Install the **playit agent** on the host (as a program/service, not the server plugin) and link it to your playit account.
2. In the playit dashboard, create a **Minecraft Java** tunnel pointing to local `127.0.0.1:25565`.
3. Give Java players the tunnel address playit shows (for example `something.tun.ply.gg`).

Keep the agent running. Enable it as a service so it starts on boot.

## Part 6 — Bedrock and console access via MCXboxBroadcast

This is what lets Bedrock players, including PS5 and Xbox, join without a port forward: the server is broadcast over Xbox Live and players join from the in-game **Friends** tab. Project: [MCXboxBroadcast](https://github.com/MCXboxBroadcast/Broadcaster).

1. **Make an alt Microsoft/Xbox account.** Create it at signup.live.com, then sign into xbox.com once to generate its gamertag and accept the Xbox terms. Note the gamertag.
2. **Install the extension.** Download `MCXboxBroadcastExtension.jar` from the MCXboxBroadcast releases and place it in `plugins/Geyser-Spigot/extensions/` (create the folder if needed). Make sure its owner matches the rest of the Geyser files so the extension can write its token cache.
3. **Authenticate.** Restart the server. The log prints a line like `open microsoft.com/link and enter code XXXXXXXX`. Open that URL, sign in as the **bot account**, and enter the code.
4. **Confirm it is healthy.** A good log shows `Successfully authenticated as <bot> with 0/1000 friends` (a real number, not `-1`), `NetherNet signaling started`, and `Creation of Xbox LIVE session was successful`.

**How players join:** they add the bot gamertag as a friend in Minecraft (Friends tab → Add Friend), the bot auto-adds them back, and the server appears in their Friends tab as a joinable session. Works for PC, mobile, Switch, Xbox, and PlayStation Bedrock clients.

## Part 7 — Server tweaks

Run these in the Crafty server console (Terminal tab):

```
whitelist off
```

Bedrock players come through Floodgate with a prefix and are not Java accounts, so the normal `whitelist add <name>` fails with "that player does not exist." Either leave the whitelist off, or add Bedrock players by their prefixed name directly in `whitelist.json`.

One person sleeping skips the night for everyone:

```
gamerule playersSleepingPercentage 0
```

## How each player joins

| Platform | How they join |
|---|---|
| Java (any) | Connect to the **playit Java tunnel** address |
| Bedrock PC / mobile / Switch | Friend the **bot gamertag**, join from the **Friends** tab (reliable behind NAT). The Warden link also works on the same LAN or with a port forward |
| Bedrock Xbox / PlayStation | Friend the **bot gamertag**, join from the **Friends** tab |

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Bedrock: "An error has occurred / Terracotta" | Generic connect failure. Confirm Geyser started on 19132, `transport: nethernet`, and the client is on a current version |
| ICE state goes to `checking` then `closed` after ~15s | WebRTC media never reached the server. Behind NAT without a port forward the direct path will not complete; use the MCXboxBroadcast (Friends tab) path instead |
| `assisted connection check from ... failed ... Check UDP port 19132` | Warden cannot reach your server from the internet (no port forward). Expected on this setup; rely on the Friends-tab path |
| MCXboxBroadcast `HTTP connect timed out`, friend count `-1` | Broken IPv6 on the host. Disable IPv6 (Part 4) |
| MCXboxBroadcast `Failed to relay offer to 127.0.0.1:19132: connection refused` | Geyser has no local signaling listener. Set `signaling.mode: hybrid` (Part 2) |
| Bedrock player: "not whitelisted" | `whitelist off`, or add them to `whitelist.json` by prefixed name |
| Java works but Bedrock does not | Check Floodgate is installed and `auth-type: floodgate` |

## Making it survive a reboot

- **Container:** `restart: always` is set, so Crafty and the server come back.
- **Firewall:** ufw rules persist.
- **Clock:** `timedatectl set-ntp true` persists.
- **IPv6:** only persists if you added the `/etc/sysctl.d` file (Part 4).
- **playit agent:** enable it as a systemd service so the Java tunnel comes back.
- **MCXboxBroadcast:** its login token is cached in its extension folder, so it re-authenticates on its own.

## Security notes

- The bot is a throwaway account on purpose; treat it as disposable.
- Keep the Warden operator link (`plugins/Geyser-Spigot/nethernet/operator-link.txt`) private.
- Change the Crafty admin password from the generated default.

## Credits

- [GeyserMC](https://geysermc.org) — Geyser and Floodgate
- [Warden](https://warden.cloud) by Ziax — NetherNet external signaling
- [playit.gg](https://playit.gg) — tunneling
- [MCXboxBroadcast](https://github.com/MCXboxBroadcast/Broadcaster) — Xbox Live session broadcasting
- [Crafty Controller](https://craftycontrol.com) — server management

## License

MIT — see [LICENSE](LICENSE).
