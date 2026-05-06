# Routing model

There are two layers:

1. `sing-box` is the outer travel tunnel.
2. Forti/openfortivpn is the work VPN inside that outer tunnel.

## Without Forti

With only `sing-box` TUN running:

```text
ordinary app traffic -> sing-box TUN -> VLESS server -> internet
```

`curl https://ifconfig.me` should show the proxy server IP.

## With Forti

`vpn-tun` starts `openfortivpnx` or `openfortivpn` with:

```text
HTTPS_PROXY=http://127.0.0.1:10809
```

So the Forti transport is:

```text
openfortivpn -> local sing-box HTTP proxy -> VLESS server -> Forti gateway
```

After Forti is up, work traffic is:

```text
work app -> ppp0 -> openfortivpn -> Forti transport -> VLESS server -> Forti gateway -> corporate network
```

This means the work packet logically goes through `ppp0`, while the physical
transport to Forti still goes through the VLESS server.

## Public Internet While Forti Is Up

Forti pushes broad/default routes in the baseline `vpn` setup too. Therefore
some ordinary public internet traffic may go through `ppp0` and exit through
the corporate/Forti egress.

Observed baseline:

```text
vpn up -> curl https://ifconfig.me -> Forti/corporate egress IP
```

`vpn-tun` preserves this Forti route policy. It changes only how the Forti
transport reaches the gateway.

## Expected IP Checks

- `sing-box` TUN only: `ifconfig.me` shows the VLESS server IP.
- `vpn` only: `ifconfig.me` shows the Forti/corporate egress IP.
- `sing-box` TUN + Forti over sing-box: `ifconfig.me` may show the
  Forti/corporate egress IP because Forti routes can win for that destination.
