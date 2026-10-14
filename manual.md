# Odoo 18 desde cero

Guion para levantar Odoo 18 a mano (sin Docker) en Ubuntu 24.04, detrás de nginx y con HTTPS. Cada paso indica dónde se ejecuta, el comando y una línea de porqué.

**Punto de partida:** la máquina de `sistema.md`: usuario propio con `sudo` y acceso SSH solo por clave con el alias `vps`.

**Ficheros de configuración:** están en las carpetas `odoo/`, `systemd/` y `nginx/` de este repositorio. En cada paso se crea el fichero en el servidor con `sudoedit` y se pega su contenido, con los valores ya sustituidos.

**Versiones de referencia:** Ubuntu 24.04 · PostgreSQL 16 · Python 3.12 · Odoo 18.0 · uv 0.12 · wkhtmltox 0.12.6.1-3 · nginx 1.24 · certbot 2.9.

**Valores que hay que sustituir:**

| Valor | Dónde aparece | De dónde sale |
|---|---|---|
| `DOMINIO` | `nginx/odoo.http`, `nginx/odoo`, pasos 8 y 9 | El subdominio que apunte a la máquina |
| `NOMBRE_BASE` | `odoo/odoo.conf`, pasos 3 y 4 | El nombre que se elija para la base (se crea en el paso 4) |
| `IDIOMA` | Paso 4 | Código del idioma, con el formato `es_ES` (español de España) |
| `HASH_CONTRASEÑA_MAESTRA` | `odoo/odoo.conf` | Se calcula en el paso 3. **No** va a este repositorio |
| IPv4 e IPv6 de la máquina | Paso 8 | `ip -4 address show dev eth0` e `ip -6 address show dev eth0 scope global` |

**No iniciar sesión en Odoo por `http`** antes del paso 9: la contraseña viajaría en claro.

---

## 1. PostgreSQL y el usuario de Odoo

**Dónde:** VPS.

```
sudo apt update
sudo apt install postgresql
```

```
sudo useradd --system --user-group --create-home --home-dir /var/lib/odoo --shell /usr/sbin/nologin odoo
```

Usuario de sistema `odoo`, con grupo propio y sin shell. `/var/lib/odoo` será el directorio de datos.

```
sudo -u postgres createuser --createdb odoo
```

Rol `odoo`, solo con permiso para crear bases. Se llama igual que el usuario de Linux para que funcione la autenticación `peer`.

## 2. Código de Odoo y entorno Python

**Dónde:** VPS.

```
sudo apt install git
sudo mkdir /opt/odoo
sudo git clone --depth 1 --branch 18.0 https://github.com/odoo/odoo.git /opt/odoo/src
sudo git -C /opt/odoo/src log -1 --oneline
```

**Entorno virtual para Odoo.** Hay varias herramientas para crearlo (`python3 -m venv`, `virtualenv`, `uv`…); aquí se usa `uv`, de Astral.

*Instalación de `uv`:*

```
curl -LsSf https://astral.sh/uv/install.sh | sudo env UV_INSTALL_DIR=/usr/local/bin UV_NO_MODIFY_PATH=1 sh
```

`uv` para todo el sistema en `/usr/local/bin` (sin las variables, acabaría en `/root/.local/bin`). No se actualiza con `apt`: `sudo uv self update`.

*Creación del entorno virtual para Odoo:*

```
sudo uv venv --python /usr/bin/python3 /opt/odoo/venv
```

**Instalación de las dependencias de Odoo (`requirements.txt`).** Algunas librerías de la lista (`psycopg2`, `python-ldap`) no se descargan ya compiladas: `uv` las compila desde su código en C durante la instalación. Para eso necesita antes un compilador y las cabeceras (`-dev`) de las librerías del sistema a las que se conectan; sin ellos, la instalación de `requirements.txt` falla.

```
sudo apt install gcc-x86-64-linux-gnu binutils libpython3.12-dev libldap-dev libsasl2-dev libpq-dev
```

| Paquete | Qué aporta |
|---|---|
| `gcc-x86-64-linux-gnu` | El compilador de C |
| `binutils` | El ensamblador (`as`) y el enlazador |
| `libpython3.12-dev` | `Python.h` |
| `libldap-dev`, `libsasl2-dev` | Cabeceras de LDAP y SASL, para compilar `python-ldap` |
| `libpq-dev` | `libpq-fe.h`, para compilar `psycopg2` |

```
sudo uv pip install --python /opt/odoo/venv/bin/python -r /opt/odoo/src/requirements.txt phonenumbers
```

Librerías de Odoo dentro de su entorno. `phonenumbers` no está en `requirements.txt`, pero la pide `account_peppol`, que se instala con Ventas o Facturación.

Si falla por un fichero que falta: `sudo apt install apt-file`, `sudo apt-file update` y `apt-file search --regexp '/NOMBRE_DEL_FICHERO$'` (el nombre sale del mensaje de error).

## 3. Configuración de Odoo

**Dónde:** VPS.

```
sudo mkdir /etc/odoo
sudo touch /etc/odoo/odoo.conf
sudo chown root:odoo /etc/odoo/odoo.conf
sudo chmod 640 /etc/odoo/odoo.conf
```

El fichero se crea vacío y se protege **antes** de escribir la contraseña maestra.

```
/opt/odoo/venv/bin/python -c "from getpass import getpass; from passlib.context import CryptContext; print(CryptContext(schemes=['pbkdf2_sha512'], pbkdf2_sha512__rounds=600000).hash(getpass()))"
```

Hash de la contraseña maestra (generada antes con `openssl rand -base64 24` y guardada en el gestor). `getpass()` la pide sin mostrarla ni dejarla en el historial. La salida empieza por `$pbkdf2-sha512$600000$`.

```
sudoedit /etc/odoo/odoo.conf
```

Pegar el contenido de [`odoo/odoo.conf`](odoo/odoo.conf) y sustituir `HASH_CONTRASEÑA_MAESTRA` por la salida del comando anterior (la línea que empieza por `$pbkdf2-sha512$`), sin comillas, y `NOMBRE_BASE` por el nombre elegido para la base.

| Línea | Por qué |
|---|---|
| `http_interface = 127.0.0.1` | Solo local: a Odoo se llega a través de nginx o del túnel |
| `addons_path` | Dónde están los módulos |
| `data_dir` | *Filestore* y sesiones en `/var/lib/odoo` |
| `admin_passwd` | Contraseña maestra del gestor de bases, en hash |
| `list_db = False` | Gestor de bases cerrado desde el principio |
| `dbfilter = ^NOMBRE_BASE$` | Odoo solo usa esa base y no muestra selector |
| `proxy_mode = True` | Odoo se fía de las cabeceras `X-Forwarded-*` de nginx |

## 4. Base de datos Odoo

**Terminal 1 (VPS):**

```
sudo -u odoo /opt/odoo/venv/bin/python /opt/odoo/src/odoo-bin -c /etc/odoo/odoo.conf -d NOMBRE_BASE -i base --without-demo=all --load-language=IDIOMA --stop-after-init
```

`-i base` instala el módulo mínimo, `--without-demo=all` sin datos de demostración, `--load-language` carga las traducciones del idioma elegido y `--stop-after-init` termina al acabar. La base nace con el usuario `admin`, contraseña `admin`.

**Cambiar la contraseña de `admin` antes de exponer Odoo a internet.** Se levanta Odoo a mano y se entra desde el equipo local por un túnel SSH.

**Terminal 1 (VPS):**

```
sudo -u odoo /opt/odoo/venv/bin/python /opt/odoo/src/odoo-bin -c /etc/odoo/odoo.conf
```

**Terminal 2 (equipo local):**

```
ssh -N -L 10000:127.0.0.1:8069 vps
```

Túnel: el 10000 del equipo local lleva al 8069 del VPS sin abrirlo a internet. `-N`: solo túnel, sin shell.

**Navegador del equipo local:** `http://localhost:10000` → entrar con `admin` / `admin` → avatar → *My Profile* → *Account Security* → *Change Password*. La nueva, guardada antes en el gestor de contraseñas. Si la interfaz sigue en inglés: avatar → *Preferences* → *Language*.

**Terminal 1:** `Ctrl+C`.

**Antes del paso 7**, que expone Odoo: cerrar sesión y confirmar que `admin` / `admin` ya no entra.

## 5. Odoo como servicio

**Dónde:** VPS.

```
sudoedit /etc/systemd/system/odoo.service
sudo systemctl daemon-reload
sudo systemd-analyze verify /etc/systemd/system/odoo.service
sudo systemctl enable --now odoo
sudo chmod -R g-w,o-rwx /var/lib/odoo
```

Contenido: [`systemd/odoo.service`](systemd/odoo.service). `Wants=` y no `Requires=`: si PostgreSQL se para, Odoo sigue vivo y se reconecta solo. `UMask=0027`: lo que crea Odoo nace sin acceso para los demás; el `chmod` corrige lo creado en el paso 4, antes de que existiera.

## 6. Informes PDF

**Dónde:** VPS.

```
cd ~
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo apt install ./wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo systemctl restart odoo
```

Build oficial con Qt parcheado: con el `wkhtmltopdf` de Ubuntu, los PDF salen sin cabecera ni pie.

**Navegador (por el túnel):** Ajustes → Técnico → Parámetros del sistema → crear `report.url` = `http://127.0.0.1:8069`. wkhtmltopdf pide estilos y fuentes al propio VPS; sin este parámetro usaría `web.base.url`, que puede no ser alcanzable desde dentro.

## 7. Nginx

**Dónde:** VPS.

**Instalar nginx.**

```
sudo apt install nginx
```

**Desactivar el sitio de bienvenida.** Se borra solo su enlace en `sites-enabled`; el fichero sigue en `sites-available`.

```
sudo rm /etc/nginx/sites-enabled/default
```

**Crear el sitio de Odoo.** Pegar el contenido de [`nginx/odoo.http`](nginx/odoo.http) y sustituir `DOMINIO`.

```
sudoedit /etc/nginx/sites-available/odoo
```

**Activar el sitio.**

```
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/odoo
```

**Validar y recargar.** `nginx -t` siempre antes de recargar: si la configuración tiene un error, lo dice sin tirar el servicio.

```
sudo nginx -t
sudo systemctl reload nginx
```

## 8. DNS

**Dónde:** panel de Cloudflare y equipo local.

| Tipo | Nombre | Contenido | Proxy |
|---|---|---|---|
| `A` | subdominio de `DOMINIO` | IPv4 de la máquina | **DNS only** |
| `AAAA` | subdominio de `DOMINIO` | IPv6 de la máquina | **DNS only** |

DNS only para que el único proxy sea nginx.

## 9. HTTPS

**Dónde:** VPS.

```
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d DOMINIO
sudo certbot renew --dry-run
```

certbot desde los repositorios de Ubuntu, sin snap. Pide el certificado (Let's Encrypt valida el dominio por el puerto 80) y edita el sitio. El simulacro de renovación debe acabar en `all simulated renewals succeeded`; la renovación real la lanza `certbot.timer`.

**Sitio final.** Reemplazar todo el contenido por el de [`nginx/odoo`](nginx/odoo) y sustituir `DOMINIO`.

```
sudoedit /etc/nginx/sites-available/odoo
sudo nginx -t
sudo systemctl reload nginx
```

Al actualizar nginx a 1.25.1 o posterior: el parámetro `http2` de los `listen` pasa a la directiva `http2 on;`.

## Secretos

| Secreto | Dónde vive | En este repo |
|---|---|---|
| Contraseña maestra | En claro, solo en el gestor de contraseñas. En `/etc/odoo/odoo.conf` (`root:odoo 640`), solo su hash | **No** |
| Contraseña del administrador de la base | Hash en `res_users`; en claro, solo en el gestor | **No** |
| Clave privada del certificado | `/etc/letsencrypt/live/DOMINIO/privkey.pem` (solo root) | **No** |
