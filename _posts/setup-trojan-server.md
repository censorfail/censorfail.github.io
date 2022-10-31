# Setup Trojan Proxy Server with Caddy

We will need the following applications/software to setup a Trojan Proxy Server with Caddy:
- **Caddy2** as the web server 
- **Trojan-Go** as the Trojan Proxy Server
- **Crontab** for scheduled tasks

## Caddy

### Install
Refer to [Caddy Documentation](https://caddyserver.com/docs/install#debian-ubuntu-raspbian)

### Configuration
Caddy file: `/etc/caddy/Caddyfile`

A sample `Caddyfile`

```
{
    # http_port 80
    https_port 8443
}

http://server.example.com {
    # bind 127.0.0.1
    templates {
        mime application/json
    }
    # header Content-Type text/plain
    header Content-Type application/json
    respond * 404 {
        body "pageok"
    }
}

https://server.example.com {
    bind 127.0.0.1
    templates {
        mime application/json
    }
    # header Content-Type text/plain
    header Content-Type application/json
    respond * 404 {
        body "pageok"
    }
}
```

## Trojan-go

### Compile and install 
Refer to [Trojan-Go Documentation](https://p4gefau1t.github.io/trojan-go/developer/build/)

For MySQL TLS support, build from [TunnelWork/trojan-go](https://github.com/TunnelWork/trojan-go) instead.

### Configuration 
See [Trojan-Go Documentation](https://p4gefau1t.github.io/trojan-go/basic/full-config/)

Point `fallback_addr` to local and `fallback_port` to `8443` in order to have a behavior closer to real TLS: returning 400 Bad Request when the client is not using TLS.

Add to systemd by creating `/lib/systemd/system/trojan-go.service` with the following content:

```
[Unit]
Description=Trojan-Go - An unidentifiable mechanism that helps you bypass GFW
Documentation=https://p4gefau1t.github.io/trojan-go/
After=network.target nss-lookup.target

[Service]
User=root
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ExecStart=/usr/bin/trojan-go -config /etc/trojan-go/config.json
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
```

Then run 

```
sudo systemctl enable trojan-go
sudo systemctl start trojan-go
```

## Crontab 

### Install

```
sudo apt install cron
```

### Config

```
sudo crontab -e
```

Append the line below to the end of file:
```
0 0 * * * /etc/trojan-go/get_latest_tls_cert.sh
```

Where `/etc/trojan-go/get_latest_tls_cert.sh` has the following content:

```
#!/bin/bash

### This should run daily to make sure the certificate is always latest ###

CERT_PATH="/var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory"
DOMAIN="server.example.com"

# Get old hash
OLD_HASH=$(sha256sum /etc/trojan-go/tls.crt | cut -d " " -f 1)
NEW_HASH=$(sha256sum ${CERT_PATH}/${DOMAIN}/${DOMAIN}.crt | cut -d " " -f 1)

if [ "${OLD_HASH}" != "${NEW_HASH}" ]; then
        echo "File changed! Start copying file and restart service after that."

        # Copy latest TLS cert
        \cp ${CERT_PATH}/${DOMAIN}/${DOMAIN}.crt /etc/trojan-go/tls.crt
        \cp ${CERT_PATH}/${DOMAIN}/${DOMAIN}.key /etc/trojan-go/tls.key

        # Restart service
        /bin/systemctl restart trojan-go
fi
```

It copies the latest TLS cert to `/etc/trojan-go/tls.crt` and `/etc/trojan-go/tls.key` and restarts the service.