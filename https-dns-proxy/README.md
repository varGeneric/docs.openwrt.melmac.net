<!-- markdownlint-disable MD013 -->

<!-- markdownlint-disable MD030 -->

# DNS Over HTTPS Proxy (https-dns-proxy)

## Description

A lean RFC8484-compatible (no JSON API support) DNS-over-HTTPS (DoH) proxy service which supports DoH servers ran by AdGuard, CleanBrowsing, Cloudflare, Google, ODVR (nic.cz) and Quad9. Based on [@aarond10](https://github.com/aarond10)'s [https-dns-proxy](https://github.com/aarond10/https_dns_proxy).

## Features

-   [RFC8484](https://tools.ietf.org/html/rfc8484)-compatible DoH Proxy.
-   Compact size.
-   Web UI (`luci-app-https-dns-proxy`) available.
-   (By default) automatically updates DNSMASQ settings to use DoH proxy when it's started and reverts to old DNSMASQ resolvers when DoH proxy is stopped.

## Screenshots (luci-app-https-dns-proxy)

![screenshot](https://docs.openwrt.melmac.net/https-dns-proxy/screenshots/screenshot01.png "https-dns-proxy screenshot")

## Requirements

This proxy requires the following packages to be installed on your router: `libc`, `libcares`, `libcurl`, `libev`, `ca-bundle`. They will be automatically installed when you're installing `https-dns-proxy`.

## Unmet Dependencies

If you are running a development (trunk/snapshot) build of OpenWrt on your router and your build is outdated (meaning that packages of the same revision/commit hash are no longer available and when you try to satisfy the [requirements](#requirements) you get errors), please flash either current OpenWrt release image or current development/snapshot image.

## How To Install

Install `https-dns-proxy` and `luci-app-https-dns-proxy` packages from Web UI or run the following in the command line:

```sh
opkg update; opkg install https-dns-proxy luci-app-https-dns-proxy;
```

## Default Settings

Default configuration has service enabled and starts the service with Google and Cloudflare DoH servers. In most configurations, you will keep the default `DNSMASQ` service installed to handle requests from devices in your local network and point `DNSMASQ` to use `https-dns-proxy` for name resolution.

By default, the service will intelligently override existing `DNSMASQ` servers settings on start to use the DoH servers and restores original `DNSMASQ` servers on stop. See the [Configuration Settings](#configuration-settings) section below for more information and how to disable this behavior.

## Configuration Settings

Configuration contains the general (named) "main" config section where you can configure which `DNSMASQ` settings the service will automatically affect and the typed (unnamed) https-dns-proxy instance settings. The original config file is included below:

```text
config main 'config'
  option update_dnsmasq_config '*'
  option force_dns '1'
  list force_dns_port '53'
  list force_dns_port '853'

config https-dns-proxy
  option bootstrap_dns '8.8.8.8,8.8.4.4'
  option resolver_url 'https://dns.google/dns-query'
  option listen_addr '127.0.0.1'
  option listen_port '5053'
  option user 'nobody'
  option group 'nogroup'

config https-dns-proxy
  option bootstrap_dns '1.1.1.1,1.0.0.1'
  option resolver_url 'https://cloudflare-dns.com/dns-query'
  option listen_addr '127.0.0.1'
  option listen_port '5054'
  option user 'nobody'
  option group 'nogroup'
```

### General Settings

#### update_dnsmasq_config

The `update_dnsmasq_config` option can be set to dash (set to `'-'` to not change `DNSMASQ` server settings on start/stop), can be set to `'*'` to affect all `DNSMASQ` instance server settings or have a space-separated list of `DNSMASQ` instances or named sections to affect (like `'0 4 5'` or `'0 backup_dns 5'`). If this option is omitted, the default setting is `'*'`. When the service is set to update the DNSMASQ servers setting on start/stop, it does not override entries which contain either `#` or `/`, so the entries like listed below will be kept in use:

```test
  list server '/onion/127.0.0.1#65453'
  list server '/openwrt.org/8.8.8.8'
  list server '/pool.ntp.org/8.8.8.8'
  list server '127.0.0.1#15353'
  list server '127.0.0.1#55353'
  list server '127.0.0.1#65353'
```

#### force_dns

The `force_dns` setting is used to force the router's default resolver to all connected devices even if they are set to use other DNS resolvers or if other DNS resolvers are hardcoded in connected devices' settings. You can additionally control which ports the `force_dns` setting should be actvive on, the default values are `53` (regular DNS) and `853` (DNS over TLS). If the listed port is open/active on OpenWrt router, the service will create a `redirect` to the indicated port number, otherwise the service will create a `REJECT` rule.

### Instance Settings

The https-dns-proxy instance settings are:

| Parameter               | Type             | Default     | Description                                                                                                                                                                                                                            |
| ----------------------- | ---------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bootstrap_dns           | IP Address       |             | The non-encrypted DNS servers to be used to resolve the DoH server name on start.                                                                                                                                                      |
| dscp_codepoint          | Integer [0-63]   |             | Optional DSCP codepoint [0-63] to set on upstream DNS server connections.                                                                                                                                                              |
| listen_addr             | IP Address       | 127.0.0.1   | The local IP address to listen to requests.                                                                                                                                                                                            |
| listen_port             | Port             | 5053 and up | If this setting is omitted, the service will start the first https-dns-proxy instance on port 5053, second on 5054 and so on.                                                                                                          |
| logfile                 | Filepath         |             | Full filepath to the file to log the instance events to.                                                                                                                                                                               |
| polling_interval        | Integer [5-3600] | 120         | Optional polling interval of DNS servers.                                                                                                                                                                                              |
|                         |                  |             |                                                                                                                                                                                                                                        |
| resolver_url            | URL              |             | The https URL to the RFC8484-compatible resolver.                                                                                                                                                                                      |
| proxy_server            | URL              |             | Optional HTTP proxy. e.g. socks5://127.0.0.1:1080. Remote name resolution will be used if the protocol supports it (http, https, socks4a, socks5h), otherwise initial DNS resolution will still be done via the bootstrap DNS servers. |
| user                    | String           | nobody      | Local user to run instance under.                                                                                                                                                                                                      |
| group                   | String           | nogroup     | Local group to run instance under.                                                                                                                                                                                                     |
| use_http1               | Boolean          | 0           | If set to 1, use HTTP/1 on installations with broken/outdated `curl` package. Included for posterity reasons, you will most likely not ever need it on OpenWrt.                                                                        |
| verbosity               | Integer          | 0           | Logging verbosity level. Fatal = 0, error = 1, warning = 2, info = 3, debug = 4.                                                                                                                                                       |
| use_ipv6_resolvers_only | Boolean          | 0           | If set to 1, forces IPv6 DNS resolvers instead of IPv4.                                                                                                                                                                                |

Please also refer to the [Usage section at upstream README](https://github.com/aarond10/https_dns_proxy/blob/master/README.md#usage) which may contain additional/more details on some parameters.

## Thanks

This OpenWrt package wouldn't have been possible without [@aarond10](https://github.com/aarond10)'s [https-dns-proxy](https://github.com/aarond10/https_dns_proxy) and his active participation in the OpenWrt package itself. Thanks to [@oldium](https://github.com/oldium) and [@curtdept](https://github.com/curtdept) for their contributions to this package. Special thanks to [@jow-](https://github.com/jow-) for general package/luci guidance.

<!-- markdownlint-disable MD033 -->

<script defer src='https://static.cloudflareinsights.com/beacon.min.js' data-cf-beacon='{"token": "911798f2c34b45338f8f8182830a3eb6"}'></script>
