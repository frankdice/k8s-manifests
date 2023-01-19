# Docker Compose Apps

These apps are meant to run outside of Bluey as they need additional privileges or similar.

## Wireguard

Wireguard is used for VPN access into the network directly. It's running on a VM on Bob at `192.168.1.2:51820` and provides access into my network. The docker-compose file automatically generates the peer configs.

`docker-compose up -d`
`/app/show-peer frankPhone`

Peer names can be found in `/etc/wireguard/wg0.conf` as well as in the docker-compose.yaml file.