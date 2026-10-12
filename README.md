# Tecnología Web I (SIS-0150) — Laboratorios

**Universidad Privada Domingo Savio** · Docente: M.Sc. Ing. Jimmy Nataniel Requena Llorentty
**Estudiante:** Leonardo Avila Bejarano · **Grupo:** CodeX

Servidor `srv-horizonte` (Ubuntu Server 24.04 con systemd) ejecutado en **Docker**, para la U.E. Horizonte.

| Lab | Tema | Verificador |
|---|---|---|
| [Lab 02](lab02) | Administración de Ubuntu Server: usuarios, grupos, setgid, SSH, script idempotente | 14/14 |
| [Lab 03](lab03) | Nginx: hosts virtuales, redirección 301, caché, autoindex, errores propios, `return 444` | 14/14 |
| [Lab 04](lab04) | Portal para padres: PHP-FPM (FastCGI), proxy inverso a una API con systemd, sesiones seguras | 16/16 |

## Estructura

```
lab02/   Dockerfile, docker-compose.yml y scripts del servidor
lab03/   configuración de Nginx (www, portal, rechazo de hosts desconocidos)
lab04/   horizonte-portal.conf, horizonte-eco.service, sesion.php, entrar.php
docs/    hojas de trabajo y bitácoras de IA (Word)
```

## Cómo levantar el servidor

```powershell
cd lab02
docker compose build
docker compose up -d
docker compose exec -u leonardo servidor bash
```

Puertos publicados: `8080→80` (HTTP), `2222→22` (SSH), `8443→443`.
Para detener sin perder datos: `docker compose stop` (no usar `down`).
