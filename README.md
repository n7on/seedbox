# seedbox

A small self-hosted seedbox stack built with Docker Compose: torrents routed
through a WireGuard VPN, with the downloads shared over SMB and streamed with
Jellyfin.

## Services

| Service | Image | Purpose | Ports |
|---|---|---|---|
| **gluetun** | `qmcgaw/gluetun` | WireGuard VPN client and kill switch | `8080` (qBittorrent web UI) |
| **qbittorrent** | `lscr.io/linuxserver/qbittorrent` | Torrent client, networked *only* through gluetun | via gluetun |
| **samba** | `dperson/samba` | SMB share of the downloads folder | `445`, `139` |
| **jellyfin** | `jellyfin/jellyfin` | Media server for the downloads | `8096`, `7359/udp`, `1900/udp` |

```
              ┌──────────── gluetun (WireGuard) ────────────┐
 internet ◄───┤  qbittorrent  ── writes ──►  /data/downloads │
              └──────────────────────────────────┬──────────┘
                                                 │
                         ┌───────────────────────┼──────────────┐
                         ▼                                      ▼
                  samba (SMB share)                    jellyfin (streaming)
```

qBittorrent uses `network_mode: service:gluetun`, so all of its traffic goes
through the VPN tunnel. If the tunnel drops, qBittorrent has no network at all.
It also waits for gluetun's healthcheck to pass before starting.

## Requirements

- Docker with the Compose plugin
- A WireGuard config from a VPN provider (one that allows P2P)
- `/dev/net/tun` on the host

## Setup

1. **Clone the repo**

   ```sh
   git clone https://github.com/n7on/seedbox.git
   cd seedbox
   ```

2. **Create the host directories** used by the bind-mounted volumes:

   ```sh
   sudo mkdir -p /data/downloads /data/stacks/seedbox/{qbittorrent,jellyfin}
   sudo chown -R 1000:1000 /data/downloads /data/stacks/seedbox
   ```

   Change these paths in `docker-compose.yml` if you want your data elsewhere.

3. **Configure the VPN.** Copy the example env file and fill in the values from
   your provider's WireGuard config:

   ```sh
   cp .env.example .env
   ```

   | Variable | Where to find it in a WireGuard `.conf` |
   |---|---|
   | `VPN_ENDPOINT_IP` | `[Peer] Endpoint` (the IP part) |
   | `VPN_ENDPOINT_PORT` | `[Peer] Endpoint` (the port part) |
   | `WIREGUARD_PUBLIC_KEY` | `[Peer] PublicKey` |
   | `WIREGUARD_PRIVATE_KEY` | `[Interface] PrivateKey` |
   | `WIREGUARD_ADDRESSES` | `[Interface] Address` |

   Also set `SMB_USER` and `SMB_PASSWORD` for the Samba share. Using the same
   login as your Jellyfin user keeps things simple. The password can't contain `;`.

4. **Start the stack**

   ```sh
   docker compose up -d
   ```

## Usage

| What | Where |
|---|---|
| qBittorrent web UI | `http://<host>:8080` |
| Jellyfin | `http://<host>:8096` |
| SMB share | `\\<host>\downloads` or `smb://<host>/downloads` (log in as `SMB_USER`) |

**qBittorrent login:** the username is `admin`. On first start the linuxserver
image prints a temporary password to the logs:

```sh
docker logs qbittorrent 2>&1 | grep -i password
```

Change it in the web UI under *Tools → Options → Web UI*.

**Check that the VPN works:** the IP below should be the VPN's, not your own.

```sh
docker exec gluetun wget -qO- https://ipinfo.io/ip
```

## Configuration notes

- `PUID`/`PGID` (default `1000`) set the owner of downloaded files.
- `TZ` is set to `Europe/Stockholm`; change it to your timezone.
- The Samba share requires the `SMB_USER` login; guest access is off. Files
  written over SMB are owned by UID/GID 1000, matching qBittorrent.
- Samba's recycle bin is disabled (`-r`), so deleting over SMB really
  deletes. Otherwise files move to a hidden `.deleted` folder that Jellyfin
  keeps listing.

## License

[MIT](LICENSE)
