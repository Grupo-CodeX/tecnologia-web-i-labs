# Laboratorio 7 — El DNS autoritativo del colegio con BIND9

BIND 9.18 como **primario (ns1)** y **secundario (ns2)** de `horizonte.internal` y de su zona inversa, con transferencia autenticada por **TSIG**, **NOTIFY**, sin recursión y con la versión oculta.

## Red `tw1-dns` (carpeta `red_dns/`)

| Nodo | IP | Función |
|---|---|---|
| ns1 | 10.10.10.2 | Primario |
| ns2 | 10.10.10.3 | Secundario |
| resolutor | 10.10.10.4 | Unbound (Lab 8) |
| cliente | 10.10.10.50 | Consultas y verificador |

```powershell
cd red_dns
docker compose -p tw1-dns build ns1
docker compose -p tw1-dns up -d ns1 ns2 resolutor cliente
docker compose -p tw1-dns exec ns1 bash          # carpeta de trabajo: /lab/lab07/practica
```

## `ns1/db.horizonte.internal` (zona directa)

```dns
$ORIGIN horizonte.internal.
$TTL 3600
@   IN  SOA ns1.horizonte.internal. ti.horizonte.internal. (
        2026100703 3600 600 1209600 300 )
@       IN  NS      ns1
@       IN  NS      ns2
ns1     IN  A       10.10.10.2
ns2     IN  A       10.10.10.3
@       IN  A       10.10.10.20
www     IN  A       10.10.10.20
portal  IN  CNAME   www
srv-horizonte IN A  10.10.10.20
@       IN  MX      10 mail
mail    IN  A       10.10.10.25
@       IN  TXT     "v=spf1 mx -all"
@       IN  CAA     0 issue "horizonte.internal"
resolutor   IN  A   10.10.10.4
impresora   IN  A   10.10.10.60
aula-lab    IN  A   10.10.10.70
biblioteca  IN  A   10.10.10.80
cafeteria   IN  A   10.10.10.81
```

## `ns1/db.10.10.10` (zona inversa)

```dns
$ORIGIN 10.10.10.in-addr.arpa.
$TTL 3600
@   IN  SOA ns1.horizonte.internal. ti.horizonte.internal. (
        2026100701 3600 600 1209600 300 )
@   IN  NS  ns1.horizonte.internal.
@   IN  NS  ns2.horizonte.internal.
2   IN  PTR ns1.horizonte.internal.
3   IN  PTR ns2.horizonte.internal.
4   IN  PTR resolutor.horizonte.internal.
20  IN  PTR www.horizonte.internal.
25  IN  PTR mail.horizonte.internal.
60  IN  PTR impresora.horizonte.internal.
70  IN  PTR aula-lab.horizonte.internal.
```

## `ns1/named.conf.options` y `ns1/named.conf.local`

```conf
options {
    directory "/var/cache/bind";
    listen-on    { any; };
    listen-on-v6 { none; };
    recursion no;                 // autoritativo: no resuelve nombres ajenos
    allow-query     { any; };
    allow-transfer  { none; };
    version "no disponible";      // oculta la versión de BIND
    dnssec-validation auto;
    minimal-responses yes;
};
```

```conf
include "/etc/bind/transferencia.key";

zone "horizonte.internal" {
    type primary;
    file "/var/lib/bind/db.horizonte.internal";
    allow-transfer { key "transferencia-horizonte"; };   // solo con TSIG
    also-notify { 10.10.10.3; };                         // avisa a ns2
    notify yes;
};

zone "10.10.10.in-addr.arpa" {
    type primary;
    file "/var/lib/bind/db.10.10.10";
    allow-transfer { key "transferencia-horizonte"; };
    also-notify    { 10.10.10.3; };
    notify yes;
};
```

## `ns2/named.conf.local`

```conf
include "/etc/bind/transferencia.key";
server 10.10.10.2 { keys { "transferencia-horizonte"; }; };

zone "horizonte.internal" {
    type secondary;
    primaries { 10.10.10.2; };
    file "/var/lib/bind/db.horizonte.internal.sec";
    allow-transfer { none; };
};

zone "10.10.10.in-addr.arpa" {
    type secondary;
    primaries { 10.10.10.2; };
    file "/var/lib/bind/db.10.10.10.sec";
    allow-transfer { none; };
};
```

> La clave `compartido/transferencia.key` la genera `instalar_ns1.sh` con `tsig-keygen -a hmac-sha256` y **no se publica** (ver `.gitignore`).

## Instalación y verificación

```bash
bash /lab/lab07/practica/instalar_ns1.sh     # en ns1
bash /lab/lab07/practica/instalar_ns2.sh     # en ns2
```
```powershell
docker compose -p tw1-dns exec cliente bash /lab/lab07/practica/verificar_lab07.sh   # 16 de 16
```

## Consultas desde el cliente

| Comando | Resultado |
|---|---|
| `dig @10.10.10.2 www.horizonte.internal` | `NOERROR`, flags `qr aa rd`, `10.10.10.20` |
| `dig @10.10.10.3 portal.horizonte.internal` | `aa`, CNAME → `www`, `10.10.10.20` |
| `dig @10.10.10.2 -x 10.10.10.25 +short` | `mail.horizonte.internal.` |
| `dig @10.10.10.2 google.com` | `REFUSED` (sin recursión) |
| `dig @10.10.10.2 horizonte.internal AXFR` | `Transfer failed.` |
| `dig -k …/transferencia.key @10.10.10.2 horizonte.internal AXFR` | zona completa (17 registros, TCP, firma TSIG) |
| `dig @10.10.10.2 CH TXT version.bind +short` | `"no disponible"` |

## NOTIFY y el serial

| Cambio | Serial | Journal de ns2 |
|---|---|---|
| `biblioteca` sin subir el serial | 2026100701 | `zone is up to date` → no copia |
| `biblioteca` con serial nuevo | 2026100702 | `Transfer completed: … 18 records … TSIG` (mismo segundo) |
| `cafeteria` sin subir el serial | 2026100702 | `zone is up to date` → ns1 y ns2 desincronizados |
| `cafeteria` con serial nuevo | 2026100703 | `Transfer completed: … 19 records` |

El secundario solo mira el **serial**: si no sube, no copia. El serial nunca debe bajar (`rndc retransfer` fuerza la copia).
