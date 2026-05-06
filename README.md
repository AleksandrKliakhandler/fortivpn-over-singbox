# Forti VPN over sing-box

Run Forti VPN through your own VLESS server using `sing-box`.

The setup has two layers:

```text
openfortivpn/openfortivpnx -> local sing-box HTTP proxy -> VLESS server -> Forti gateway
```

For ordinary traffic, `sing-box` can also run as a TUN and route traffic through
the same VLESS server.

## Files

- `bin/sing-box-tun` - starts `sing-box` in TUN mode and exposes local
  HTTP/SOCKS proxies on `127.0.0.1:10809` / `127.0.0.1:10808`.
- `bin/vpn-tun` - supervises `openfortivpnx` or `openfortivpn` through
  `http://127.0.0.1:10809`, with reconnects, status/log commands, and a macOS
  OTP/push pinentry dialog.
- `config/sing-box.tun.example.json` - safe template for your private VLESS
  config.

Real `config/*.json` files are ignored by git. Do not commit your real VLESS
server, UUID, or corporate VPN config.

## Requirements

Install:

- `sing-box`
- `openfortivpnx` or `openfortivpn`

The scripts look in `PATH` first and then in `~/bin`.

## Normal Flow

Create the real local sing-box config:

```sh
cp config/sing-box.tun.example.json config/sing-box.tun.json
$EDITOR config/sing-box.tun.json
```

Put your real VLESS `server`, `server_name`, `server_port`, and `uuid` into
`config/sing-box.tun.json`.

Start sing-box TUN:

```sh
bin/sing-box-tun
```

In another terminal, start Forti through sing-box. Pass your normal
`openfortivpn` config:

```sh
bin/vpn-tun u /path/to/openfortivpn.conf
```

Or use an environment variable:

```sh
VPN_TUN_CONFIG=/path/to/openfortivpn.conf bin/vpn-tun u
```

Check status and logs:

```sh
bin/vpn-tun status
bin/vpn-tun log
```

Stop Forti:

```sh
bin/vpn-tun d
```

Stop sing-box with `Ctrl-C` in its terminal.

## Manual Forti Run

`vpn-tun` is just one supervised wrapper with reconnects and a macOS OTP/push
dialog. You do not have to use it.

The important part is to run your Forti client through the local sing-box HTTP
proxy:

```sh
HTTPS_PROXY=http://127.0.0.1:10809 openfortivpn -c /path/to/openfortivpn.conf
```

Use whatever `openfortivpn`/`openfortivpnx` flags your own setup needs.

## Diagnostics

Check the local HTTP proxy:

```sh
curl --proxy http://127.0.0.1:10809 https://ifconfig.me
```

Check the local SOCKS proxy:

```sh
curl --socks5-hostname 127.0.0.1:10808 https://ifconfig.me
```

TUN mode should make plain `curl` show the proxy server IP:

```sh
curl https://ifconfig.me
```

Logs:

```sh
tail -n 120 /tmp/sing-box-tun.log
tail -n 120 /tmp/vpn-tun.log
```

## Notes

Use this wrapper only when the Forti transport must go through sing-box.
For direct/non-travel use, run your regular Forti client as usual.

`vpn-tun` does not rewrite Forti's pushed route policy. If Forti sends broad or
default routes, public IP checks may show the Forti/corporate egress IP while
the VPN is up. See `docs/routing.md`.

`vpn-tun` watches `ppp0` by default. Override it with `VPN_TUN_IFACE` if your
Forti client uses another interface name.

## Checks

The real sing-box configs live in this directory and are intentionally ignored
by git:

```sh
sing-box check -c config/sing-box.tun.json
```
