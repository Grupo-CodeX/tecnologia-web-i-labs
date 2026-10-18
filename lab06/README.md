# Laboratorio 6 — Radiografía DNS de dominios reales

Consultas con `dig` a dominios públicos para entender la jerarquía DNS, los tipos de registro, el SOA y la caché, y un script propio (`radiografia_dns.sh`) que diagnostica la salud DNS de un dominio.

**Entorno:** contenedor `srv-horizonte` (Ubuntu 24.04), `dig` 9.18.39 (`dnsutils`). Resolutor explícito `@1.1.1.1` en todas las consultas. Fecha de observación: 14/10/2026.

## Parte A — El viaje de una consulta

```bash
dig +trace +nodnssec +time=3 +tries=1 @1.1.1.1 www.upds.edu.bo
```

| Salto | Respondió | Qué dijo |
|---|---|---|
| 1 | `1.1.1.1` | Los 13 nombres de la raíz (`a`–`m.root-servers.net`) |
| 2 | `a.root-servers.net` | `.bo` → `ns.nic.bo`, `anycast.ns.nic.bo`, `ns.dns.br`, `ns2.nic.fr` |
| 3 | `ns.nic.bo` | `upds.edu.bo` → `ns2` a `ns6.megalink.com` |
| 4 | `ns2.megalink.com` | `www.upds.edu.bo. 300 A 35.212.117.103` |

```bash
dig +norec @ns4.megalink.com upds.edu.bo SOA   # NOERROR, flag aa
dig +norec @ns6.megalink.com upds.edu.bo SOA   # REFUSED, sin aa → delegación coja
```

El padre (`.bo`) delega en **ns2–ns6**, pero la zona declara **ns2–ns5**: `ns6` responde `REFUSED`.

## Parte B — Registros de upds.edu.bo

| Consulta | Resultado |
|---|---|
| `A` | `35.212.117.103` |
| `AAAA` | sin registro (`NOERROR`, `ANSWER: 0`) |
| `MX` | `0 upds-edu-bo.mail.protection.outlook.com.` |
| `TXT` | `v=spf1 include:spf.protection.outlook.com -all` + verificaciones |
| `_dmarc TXT` | `v=DMARC1; p=quarantine; sp=quarantine; pct=100; …` |
| `SOA` | `NS2.MegaLink.com.` · RNAME sin punto final · `2026081101 28800 7200 1209600 7200` |
| `-x 35.212.117.103` | `…bc.googleusercontent.com.` (Google Cloud) |
| `cloudflare.com HTTPS` | `1 . alpn="h3,h2" …` |
| `google.com CAA` | `0 issue "pki.goog"` |

## Parte C — TTL y caché

Dos consultas a `www.upds.edu.bo` con 5 s de diferencia: TTL **1 → 63**. La primera copia estaba por vencer; la segunda vino de otra instancia de caché de 1.1.1.1 (anycast).

## Parte D — `radiografia_dns.sh` (4 TODO)

```bash
# D1 — autoridad y serial (dentro del bucle de NS)
  if grep -q 'flags:.* aa' <<<"$r"; then
    SERIALES+="$(awk '$4=="SOA"{print $7}' <<<"$r") "
  else
    COJOS+="$ns "
  fi
```

```bash
# D2 — DMARC
DMARC=$(q "_dmarc.$DOMINIO" TXT | tr -d '"' | grep -i '^v=DMARC1' | head -1)
POL=$(grep -oE '(^|[;[:space:]])p=(none|quarantine|reject)' <<<"$DMARC" | grep -oE 'p=(none|quarantine|reject)' | head -1)
if [[ -z $DMARC ]]; then mal "DMARC: no existe"
elif [[ $POL == p=none ]]; then aviso "DMARC solo observa ($POL)"
elif [[ $POL == p=quarantine || $POL == p=reject ]]; then ok "DMARC con política $POL"
else mal "DMARC: política ausente o no reconocida"; fi
```

```bash
# D3 — CAA
CAA=$(q "$DOMINIO" CAA | tr '\n' ';')
if [[ -n $CAA ]]; then ok "CAA: $CAA"; else mal "CAA: sin registros en el nombre consultado; revisar herencia"; fi
```

```bash
# D4 — DNSSEC
DS=$(q "$DOMINIO" DS)
AD=$(dig +dnssec +time=3 +tries=1 @"$RES" "$DOMINIO" SOA | grep -c 'flags:.* ad')
if [[ -n $DS && $AD -ge 1 ]]; then ok "DNSSEC: firmado y validado (DS en $PADRE, flag ad)"
elif [[ -n $DS ]]; then mal "DNSSEC: hay DS pero no se observó validación; revisar respuesta y resolutor"
else mal "DNSSEC: no firmado según DS consultado (sin DS en $PADRE)"; fi
```

| Dominio | Puntaje | Hallazgos |
|---|---|---|
| `upds.edu.bo` | 6/12 | delegación coja (ns6), RNAME sin punto, sin AAAA, sin CAA, sin DNSSEC |
| `cloudflare.com` | 12/12 | todo correcto |
| `wikipedia.org` | 10/12 | SPF `~all`, sin DS |
| `mozilla.org` | 10/12 | SPF `~all`, sin DS |

**Verificador: 12 de 12 comprobaciones.** El puntaje de cada dominio es su diagnóstico, no la nota del laboratorio.
