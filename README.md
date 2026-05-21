# dns-geosite-proxy

DNS proxy with geosite-based routing and MikroTik firewall address-list integration.

Classifies DNS queries using [v2fly/domain-list-community](https://github.com/v2fly/domain-list-community) (`dlc.dat`),
forwards them to the appropriate upstream, and pushes resolved IPs into MikroTik
`ip/firewall/address-list` or `ipv6/firewall/address-list` via the REST API.

Designed to run as a container on MikroTik RouterOS 7.x (tested on hAP ax3, arm64).

## How it works

```
DNS client
  -> MikroTik embedded DNS (cache layer)
    -> dns-geosite-proxy :53
      -> classify domain by geosite rules
      -> forward to upstream (Yandex / Cloudflare DoH / local)
      -> return response to MikroTik
      -> push resolved IPs to MikroTik address-list (async)
```

MikroTik firewall mangle rules use the populated address-lists to mark
routing for DMZ or direct paths.

## Requirements

- Go 1.23+
- Docker with buildx (for arm64 image build)
- MikroTik RouterOS 7.4+ with container support enabled
- USB flash drive on the MikroTik (recommended, 128 MB internal flash is tight)

## Quick start

```bash
# 1. Clone and enter the project
git clone https://github.com/yourname/dns-geosite-proxy
cd dns-geosite-proxy

# 2. Download geosite database
make download-dlc

# 3. Create your config
cp config.example.json config.json
# edit config.json: set mikrotik.address, username, password

# 4. Build Docker image for arm64 and save as .tar.gz
make docker-save
# -> build/dns-geosite-proxy-arm64.tar.gz
```

## Deploy to MikroTik

Tested on hAP ax3 (RBD53G-5HaxD2HaxD), RouterOS 7.21.3, arm64, USB flash as storage.

### 1. Build and upload

```bash
# Build arm64 image and save as tar.gz
make docker-save
# -> build/dns-geosite-proxy-arm64.tar.gz

# Upload image and dlc.dat to MikroTik USB
# Note: USB mounts as usb1-part1 (partition name, not usb1)
scp build/dns-geosite-proxy-arm64.tar.gz admin@10.0.10.1:/usb1-part1/
scp data/dlc.dat admin@10.0.10.1:/usb1-part1/mounts/dns-proxy/data/dlc.dat
scp config.json admin@10.0.10.1:/usb1-part1/mounts/dns-proxy/config.json
```

> Check the actual USB mount path on your router: `/file/print` — it may be `usb1`, `usb1-part1`, or `disk1` depending on the drive.

### 2. Container package

The `container` package must be installed. Check:

```routeros
/system/package/print
```

If missing — download the matching `container-X.XX.X-arm64.npk` from mikrotik.com,
upload it, and reboot.

### 3. Create API user and enable REST API

The container accesses MikroTik REST API over HTTP. Create a dedicated user
restricted to the container IP:

```routeros
/user/add name=api group=full password=<secret> address=172.16.0.2
```

The REST API is served by the `www` service (port 80). By default `www` is
bound to the LAN subnet only. Add the container subnet to the allowed addresses:

```routeros
# Check current address restriction
/ip/service/print where name=www

# Add container subnet alongside your existing LAN (adjust 10.0.10.0/24 to yours)
/ip/service/set www address=10.0.10.0/24,172.16.0.2/32
```

> **Why not restrict to just 172.16.0.2/32?** The `www` service also serves
> WebFig. Removing your LAN subnet will lock you out of the web interface.

In `config.json` set:
```json
{
  "address": "http://10.0.10.1",
  "username": "api",
  "password": "<secret>"
}
```

### 4. Container network

```routeros
# veth interface for the container (/30 = 2 usable IPs)
/interface/veth/add name=veth-dns address=172.16.0.2/30 gateway=172.16.0.1

# Assign the router side of the pair
/ip/address/add address=172.16.0.1/30 interface=veth-dns
```

### 5. NAT

Allow the container to reach external DNS upstreams (77.88.8.8, 1.1.1.1, etc.):

```routeros
/ip/firewall/nat/add \
    chain=srcnat \
    action=masquerade \
    src-address=172.16.0.2 \
    out-interface-list=wan \
    comment=dns-proxy:masquerade
```

### 6. Firewall

Two rules are needed. The default MikroTik firewall drops input traffic not
coming from the LAN interface list — `veth-dns` is not in that list, so
without an explicit rule the container cannot reach the REST API.

```routeros
# Container → MikroTik REST API on port 80 (www service)
# Must be placed before the "drop all not coming from LAN" rule
/ip/firewall/filter/add \
    chain=input \
    action=accept \
    in-interface=veth-dns \
    protocol=tcp \
    dst-port=80 \
    connection-state=new \
    comment=dns-proxy:api \
    place-before=[find comment="defconf: drop all not coming from LAN"]

# Container → internet for upstream DNS queries (77.88.8.8, 1.1.1.1, etc.)
# Must be placed before the "drop all from WAN not DSTNATed" rule
/ip/firewall/filter/add \
    chain=forward \
    action=accept \
    in-interface=veth-dns \
    connection-state=new \
    comment=dns-proxy:out \
    place-before=[find comment="defconf: drop all from WAN not DSTNATed"]
```

### 7. Container global config

Where to store image layers and pull cache (set once):

```routeros
/container/config/set \
    layer-dir=/usb1-part1/docker/layers \
    tmpdir=/usb1-part1/docker/pull \
    memory-high=256MiB
```

### 8. Mount points

```routeros
# Config file (read-only)
/container/mounts/add \
    list=dns-proxy \
    comment=dns-config \
    src=/usb1-part1/mounts/dns-proxy/config.json \
    dst=/etc/dns-proxy/config.json \
    read-only=yes

# Data directory: dlc.dat + auto-update target (read-write)
/container/mounts/add \
    list=dns-proxy \
    comment=dns-data \
    src=/usb1-part1/mounts/dns-proxy/data \
    dst=/data
```

### 9. Environment variables

```routeros
/container/envs/add list=time_zone key=TZ value=Europe/Moscow
```

### 10. Create and start container

```routeros
/container/add \
    file=usb1-part1/dns-geosite-proxy-arm64.tar.gz \
    interface=veth-dns \
    root-dir=/usb1-part1/containers/dns-proxy \
    layer-dir=/usb1-part1/docker/layers \
    mountlists=dns-proxy \
    envlists=time_zone \
    workdir=/app \
    start-on-boot=yes
```

Wait for extraction (status goes `extracting` → `stopped`):

```routeros
/container/print
```

Then start:

```routeros
/container/start 0
```

### 11. Verify

Check container logs (Container → Log tab in Winbox, or):

```routeros
/log/print where topics~"container"
```

Expected startup output:
```
[entrypoint] Starting dns-geosite-proxy...
[INFO]  dns-geosite-proxy v0.1.0 (commit: abc1234, built: 2026-03-09T...)
[INFO]  geosite: loaded 1420 categories from /data/dlc.dat
[INFO]  listening on :53 (async_push=true)
```

Test DNS resolution via the container:

```routeros
:resolve server=172.16.0.2 domain-name=youtube.com
/ip/firewall/address-list/print where list=dmz_proxy
```

### 12. Routing via dmz

At this point the container resolves domains and pushes IPs into `dmz_proxy`
address-list. To actually route that traffic through a DMZ gateway, add a
mangle rule and a routing table entry.

```routeros
# Mark packets destined to dmz_proxy addresses (from LAN) with a routing mark
/ip/firewall/mangle/add \
    action=mark-routing \
    chain=prerouting \
    dst-address-list=dmz_proxy \
    in-interface-list=lan \
    new-routing-mark=dmz-route \
    comment=dns-proxy:dmz-route

# Route marked packets through the dmz gateway
# Replace "dmz" with your actual DMZ interface or gateway IP
/ip/route/add \
    gateway=dmz \
    routing-table=dmz-route
```

> `gateway=dmz` is the name or IP of your DMZ tunnel interface (e.g. `odmz-client1`,
> `wireguard1`, or an IP like `10.8.0.1`). Adjust to match your setup.

After adding these rules, traffic to any IP in `dmz_proxy` from LAN clients
will be routed through the DMZ. Verify with:

```routeros
/ip/firewall/mangle/print
/ip/route/print where routing-table=dmz-route
/ip/firewall/address-list/print where list=dmz_proxy
```

## Configuration

See `config.example.json` for a fully annotated example. Key sections:

### dns.servers

Rules evaluated top-to-bottom; first match wins. The server with `"fallback": true`
catches everything not matched by earlier rules.

Rule prefix syntax (same as xray/v2ray):

| Prefix | Example | Match |
|---|---|---|
| `geosite:` | `geosite:category-ru` | dlc.dat category lookup |
| `full:` | `full:example.com` | exact FQDN only |
| `domain:` | `domain:example.com` | domain + subdomains |
| `keyword:` | `keyword:tracker` | substring anywhere |
| `regexp:` | `regexp:.*\.ru$` | Go regexp |
| _(none)_ | `example.com` | same as `domain:` |

Tags `direct`, `proxy`, `block` are built-in conventions.
`block` returns NXDOMAIN without querying any upstream.

### mikrotik.address_lists

Maps routing tags to MikroTik address-list names with TTL policy:

```json
{
  "proxy": {
    "list": "dmz_routes",
    "ttl": "336h",
    "refresh": "72h"
  }
}
```

An IP is added with `ttl` on first resolution. On subsequent resolutions
the timeout is refreshed only if the remaining time is below `refresh`.
This avoids hammering the REST API on frequently-visited domains.

## Development

```bash
make build-local    # build for local arch
make test           # run unit tests
make lint           # golangci-lint (install separately)
make check-deps     # show available module updates
make update-deps    # apply updates
make vuln-check     # govulncheck CVE scan
```

## dlc.dat updates

Inside the container, crond runs `update-dlc.sh` every Sunday at 03:00.
After download it sends `SIGHUP` to `dns-proxy`, which reloads the geosite
database in-place without restarting or dropping DNS service.

Manual update from outside:
```bash
docker exec dns-geosite-proxy /app/update-dlc.sh
```

Or rebuild the database locally and copy to the router:
```bash
make download-dlc
scp data/dlc.dat admin@192.168.88.1:/usb1/data/
# then send SIGHUP inside container or restart it
```

## Project structure

```
dns-geosite-proxy/
├── src/
│   ├── main.go              signal handling (SIGHUP=reload, SIGTERM=exit)
│   ├── config/config.go     JSON config with Duration type for "336h" strings
│   ├── classifier/          domain -> tag + upstream (pre-compiled rules)
│   ├── geosite/loader.go    dlc.dat decoder via protowire (no codegen)
│   ├── dns/server.go        miekg/dns handler, UDP + TCP, TC retry
│   └── mikrotik/
│       ├── client.go        REST API client + FormatTimeout/ParseTimeout
│       └── addresslist.go   EnsureEntry: add / refresh / skip logic
├── docker/
│   ├── Dockerfile           multi-stage: builder(host arch) + runtime(arm64)
│   ├── entrypoint.sh        init: download dlc.dat -> crond -> exec dns-proxy
│   └── update-dlc.sh        curl + sanity check + SIGHUP
├── config.example.json
├── Makefile
└── LICENSE
```

## License

MIT - see [LICENSE](LICENSE).
