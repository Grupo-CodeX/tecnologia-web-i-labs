# Laboratorio 1 — El viaje de una petición HTTP

Objetivo: entender qué ocurre entre el navegador y el servidor, usando `curl`, DevTools y el servidor de práctica `servidor_eco.py` (Python, puerto 8080).

## Iniciar el servidor de práctica

```powershell
cd C:\codigo\lab01\inicio
python servidor_eco.py        # escucha en http://127.0.0.1:8080
```

## Comandos y resultados

| # | Comando | Código | Qué demuestra |
|---|---|---|---|
| 1 | `curl.exe -I http://127.0.0.1:8080/` | 200 OK | HEAD: solo cabeceras (`Content-Length: 477`, `Server: HorizonteEco/1.0`) |
| 2 | `curl.exe -i http://127.0.0.1:8080/eco` | 200 OK | El servidor devuelve en JSON el método, la ruta y las cabeceras recibidas |
| 3 | `curl.exe -i -X POST -d "nombre=Avila" http://127.0.0.1:8080/eco` | 201 Created | Envío de un cuerpo `application/x-www-form-urlencoded` |
| 4 | `curl.exe -IL http://127.0.0.1:8080/antiguo` | 301 → 200 | `Location: /`; `-L` sigue la redirección |
| 5 | `curl.exe -i http://127.0.0.1:8080/privado` | 403 Forbidden | Acceso prohibido (no pide credenciales) |
| 6 | `curl.exe -i http://127.0.0.1:8080/noexiste` | 404 Not Found | Error del cliente: el recurso no existe |
| 7 | `curl.exe -s -o NUL -w "Código: %{http_code} Tiempo: %{time_total}s\n" http://127.0.0.1:8080/lento` | 200 | `time_total` ≈ 2,01 s (el servidor espera 2 s) |

## Cookies

```powershell
curl.exe -i -c galletas.txt -b galletas.txt http://127.0.0.1:8080/cookie   # visitas=1
curl.exe -i -c galletas.txt -b galletas.txt http://127.0.0.1:8080/cookie   # visitas=2
```

`Set-Cookie: visitas=N; Path=/; HttpOnly; SameSite=Lax; Max-Age=3600`

- **HttpOnly:** JavaScript no puede leerla (protege contra XSS).
- **SameSite=Lax:** no viaja en peticiones POST iniciadas por otros sitios (protege contra CSRF).
- **Max-Age=3600:** dura una hora.

## DevTools

| Petición | Código | Tiempo |
|---|---|---|
| `/cookie` | 200 | 5 ms |
| `/antiguo` | 301 (disk cache) → 200 | 12 ms + 2 ms |
| `/lento` | 200 | 2,00 s |

## Sitio real

```powershell
curl.exe -sI https://www.google.com      # Server: gws, X-Frame-Options, CSP-Report-Only, Alt-Svc h3
curl.exe -sI https://www.upds.edu.bo     # Server: nginx + WordPress (wp-json) + PHP; sin HSTS ni CSP
```
