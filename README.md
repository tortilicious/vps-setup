# vps-setup

Guion para instalar **Odoo 18 on premise** en un servidor con Ubuntu 24.04: a mano (sin Docker), detrás de nginx y con HTTPS, empezando por la máquina recién instalada.

## Arquitectura

```
Navegador ──HTTPS:443──▶ nginx ──HTTP──▶ Odoo (127.0.0.1:8069) ──socket local──▶ PostgreSQL 16
```

Odoo y PostgreSQL solo escuchan dentro de la máquina; lo único expuesto a internet es nginx.

## Cómo usarlo

Seguir [`manual.md`](manual.md) de principio a fin, en orden:

| Paso | Qué deja hecho |
|---|---|
| 1–4 | Servidor con usuario propio, acceso solo por clave SSH y root deshabilitado |
| 5–6 | PostgreSQL, usuario `odoo`, código de Odoo y su entorno Python |
| 7–9 | Configuración, base de datos y Odoo como servicio |
| 10 | Informes PDF |
| 11–13 | nginx, DNS y HTTPS |

## Estructura del repositorio

| Ruta | Qué contiene |
|---|---|
| `manual.md` | El guion completo |
| `odoo/odoo.conf` | Configuración de Odoo |
| `systemd/odoo.service` | Unit de systemd para Odoo |
| `nginx/odoo` | Sitio de nginx con HTTPS |

Los ficheros de configuración se pegan en el servidor en el paso que indica el manual.

## Convenciones

- Los valores en MAYÚSCULAS (`DOMINIO`, `USUARIO`, `NOMBRE_BASE`…) se sustituyen por los reales. La tabla del principio de `manual.md` dice de dónde sale cada uno.
- **Ningún secreto vive en este repositorio.** El manual indica dónde vive cada uno.
