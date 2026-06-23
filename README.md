# envoy-edge — VPS raw-TCP → single HTTP/2 connection to home

Drop-in VPS edge. Accepts raw TCP and multiplexes every connection into HTTP/2
streams over **one** long-lived TCP connection to your home box — so your router
sees a single connection instead of thousands.

```
clients ──raw TCP──▶ VPS (this) ──1 HTTP/2 conn──▶ home receiver
                    (Envoy tcp_proxy tunneling)
```

## Deploy on the VPS

```bash
git clone <repo> && cd .../proxy-gateway/deploy/envoy-edge
cp .env.example .env
nano .env                 # set HOME_ADDR / HOME_PORT / LISTEN_PORT
docker compose up -d
docker compose logs -f
```

That's it — pull updates with `git pull && docker compose up -d`. Your `.env` is
gitignored, so config changes never conflict.

## How the connection count is bounded

`CONCURRENCY` (Envoy `--concurrency`) = worker threads = **max upstream TCP
connections to home** (one pool per worker). `CONCURRENCY=1` ⇒ a single connection
carries every stream. Raise to 2–4 only if one CPU can't keep up (→ that many
connections). Measured: 300 concurrent client TCP conns → **1** upstream conn at
`CONCURRENCY=1`, every byte intact.

`connection_keepalive` pings home every 30 s so NAT/the router doesn't drop the
idle long-lived connection.

## The home side

Anything that speaks **HTTP/2 cleartext (h2c)** and accepts the `POST /c` streams.
Each stream's body is one client's raw TCP bytes; write your response bytes back as
the response body. Headers carry context (`x-client-ip`, …).

A working reference receiver is `../../lab/envoy/mock/mock_gateway.py` (it echoes;
replace the per-stream handler with your demux/forward). Run it on the home box on
`HOME_PORT` and port-forward that port on your router to the home box.

## Over the public internet

This link is cleartext h2c. Before exposing home to the internet:

* uncomment the upstream `transport_socket` (TLS) in `envoy.tmpl.yaml` and serve
  TLS on the home receiver;
* add a shared `x-auth-token` header (commented in the template) and check it at
  home — otherwise anyone who finds `HOME_ADDR:HOME_PORT` can push streams.

## Tuning for load

The compose already raises the container's `nofile` ulimit and namespaced net
sysctls (`somaxconn`, `tcp_max_syn_backlog`, `ip_local_port_range`, …) for the
many inbound client connections.

A few limits live on the **host** (Docker can't set them per-container, and with
port-publish the inbound path still crosses the host's NAT/conntrack). On the VPS:

```bash
sudo tee /etc/sysctl.d/99-edge.conf >/dev/null <<'EOF'
fs.file-max = 2097152
net.netfilter.nf_conntrack_max = 1048576
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
EOF
sudo sysctl --system
# raise conntrack hashsize too if needed:
echo 262144 | sudo tee /sys/module/nf_conntrack/parameters/hashsize
```

If conntrack is the bottleneck, switch the edge to `network_mode: host` (drop the
`ports:` mapping, set the listener to `LISTEN_PORT` directly) — that removes the
NAT/conntrack path entirely, at the cost of binding the host port directly.

## Files

| file | purpose |
|---|---|
| `envoy.tmpl.yaml` | Envoy config template (`__HOME_ADDR__`/`__HOME_PORT__` filled at start) |
| `docker-compose.yml` | renders the template from `.env` and runs Envoy |
| `.env.example` | copy to `.env`, set your values |
