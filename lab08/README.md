# Laboratorio 8 — El resolutor del colegio (Unbound) y el DNS como código

**Unbound 1.19.2** como resolutor privado (10.10.10.4): acceso restringido, caché, validación **DNSSEC**, salida cifrada **DoT**, filtrado **RPZ** y `stub-zone` hacia ns1/ns2. La zona pública se gestiona con **DNSControl 5.2.0**.

```
PC / cliente ──▶ resolutor 10.10.10.4 (Unbound: caché · DNSSEC · RPZ)
                    ├─ horizonte.internal / 10.10.10.in-addr.arpa ─▶ ns1 / ns2 (stub-zone)
                    └─ resto ══ DoT :853 ══▶ 1.1.1.1 (Cloudflare) · 9.9.9.9 (Quad9)
```

## `unbound/horizonte-resolutor.conf` (fragmentos completados)

```conf
server:
    interface: 0.0.0.0
    access-control: 0.0.0.0/0 refuse
    access-control: 127.0.0.0/8 allow
    access-control: 10.10.10.0/24 allow
    hide-identity: yes
    hide-version: yes
    module-config: "respip validator iterator"
    domain-insecure: "horizonte.internal"
    domain-insecure: "10.10.10.in-addr.arpa"
    private-address: 10.0.0.0/8
    private-domain: "horizonte.internal"
    local-zone: "10.in-addr.arpa." nodefault
    tls-cert-bundle: "/etc/ssl/certs/ca-certificates.crt"

stub-zone:
    name: "horizonte.internal"
    stub-addr: 10.10.10.2
    stub-addr: 10.10.10.3

stub-zone:
    name: "10.10.10.in-addr.arpa"
    stub-addr: 10.10.10.2
    stub-addr: 10.10.10.3

forward-zone:
    name: "."
    forward-tls-upstream: yes
    forward-addr: 1.1.1.1@853#cloudflare-dns.com
    forward-addr: 9.9.9.9@853#dns.quad9.net

rpz:
    name: "rpz.horizonte.internal"
    zonefile: "/etc/unbound/rpz.horizonte.internal"
    rpz-log: yes
```

## `unbound/rpz.horizonte.internal`

```dns
$ORIGIN rpz.horizonte.internal.
apuestas-ejemplo.com           CNAME .        ; NXDOMAIN
*.apuestas-ejemplo.com         CNAME .        ; subdominios también
horizonte-notas-login.com      A 10.10.10.20  ; phishing → página de aviso
*.horizonte-notas-login.com    A 10.10.10.20
rastreo.publicidad-ejemplo.net CNAME *.       ; NODATA
```

## `dns-como-codigo/dnsconfig.js` (TODO completados)

```javascript
SOA("@", "ns1.ue-horizonte.example.", "ti.ue-horizonte.example.", 3600, 600, 1209600, 300),
DMARC_BUILDER({
    policy: "quarantine",
    rua: ["mailto:dmarc@ue-horizonte.example"]
}),
CAA_BUILDER({
    label: "@",
    iodef: "mailto:ti@ue-horizonte.example",
    issue: ["letsencrypt.org"],
    issuewild: "none"
}),
```

```bash
cd dns-como-codigo && mkdir -p zonas
dnscontrol check && dnscontrol preview && dnscontrol push
named-checkzone ue-horizonte.example zonas/ue-horizonte.example.zone   # OK, serial 2026101800
```

## Instalación y verificación

```bash
apt-get install -y unbound-anchor dns-root-data     # la imagen no traía el ancla DNSSEC
bash /lab/lab08/practica/instalar_resolutor.sh
bash /lab/lab08/practica/verificar_lab08.sh         # 17 de 17
```

## Resultados

| Prueba | Antes | Después |
|---|---|---|
| `www.horizonte.internal` sin / con `domain-insecure` | SERVFAIL | `10.10.10.20` |
| `-x 10.10.10.25` sin / con `nodefault` | NXDOMAIN (zona local) | `mail.horizonte.internal.` |
| Caché `www.bcb.gob.bo` | 996 ms | 4 ms |
| `cloudflare.com` / `dnssec-failed.org` | flag `ad` | SERVFAIL |
| RPZ | apuestas → NXDOMAIN · phishing → 10.10.10.20 · rastreo → NODATA | |

| Captura tcpdump | Destino | Contenido visible |
|---|---|---|
| Con DoT | `1.1.1.1:853`, `9.9.9.9:853` (49 paquetes) | nada |
| Sin DoT (inicio) | `1.1.1.1:53` (15 paquetes) | `A? www.upds.edu.bo.`, `DS? bo.` … |

**Cliente con DNS propio:** `docker inspect` → `["10.10.10.4"]`; `dig www.horizonte.internal` **sin `@`** → `10.10.10.20`; `getent hosts` también resuelve. Ya no hace falta el archivo `hosts`.

**DNSControl:** cambiar `IP_WEB` a `203.0.113.21` → `preview` propone 2 cambios (`@` y `www`); al volver a `.20` → `0 corrections`.
