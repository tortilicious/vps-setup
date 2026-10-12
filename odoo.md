# Odoo 18 desde cero

Runbook para levantar Odoo 18 a mano (sin Docker) en un Ubuntu 24.04, detrás de nginx y con HTTPS. Cada paso: dónde se ejecuta, el comando y una línea de porqué.

**Punto de partida:** la máquina del `README.md` (Bloque 0): usuario propio con `sudo`, acceso SSH solo por clave con el alias `vps`.

**Versiones de referencia:** Ubuntu 24.04 · PostgreSQL 16 · Python 3.12 · Odoo 18.0 · uv 0.12 · wkhtmltox 0.12.6.1-3 · nginx 1.24 · certbot 2.9.

**Valores que hay que sustituir:**

| Valor en esta guía | De dónde sale |
|---|---|
| `CONTRASEÑA_MAESTRA` | Generarla con `openssl rand -base64 24` y guardarla en el gestor de contraseñas. **Nunca** en este repositorio ni en ningún fichero de la máquina |
| `HASH_CONTRASEÑA_MAESTRA` | El hash que se calcula en el paso 3 a partir de la contraseña maestra. Tampoco va al repositorio |
| `odoo.mlozano.dev` | El subdominio que apunte a la máquina |
| `89.58.45.111` y la IPv6 | `ip -4 address show dev eth0` y `ip -6 address show dev eth0 scope global` en la máquina |
| `test-db` | El nombre de la base que se cree en el paso 4 |

> **Huecos por confirmar en la Sesión 11** (cruzando con el journal de `sudo`): el comando exacto con el que se instaló PostgreSQL y si `git` hubo que instalarlo. Están marcados con ⚠️.

---

## 1. PostgreSQL y el usuario de Odoo

**Dónde:** VPS.

⚠️ Instalar PostgreSQL (comando por confirmar):

```
sudo apt install postgresql
```

Instala el servidor y crea el clúster `16/main`, que arranca solo con la unit `postgresql@16-main.service`.

```
sudo grep -Ev '^#|^$' /etc/postgresql/16/main/pg_hba.conf
```

Solo para comprobar, **no se edita**: en Ubuntu ya viene la regla `local all all peer`, que deja entrar por socket a un usuario de Linux en el rol del mismo nombre, sin contraseña.

```
sudo useradd --system --user-group --create-home --home-dir /var/lib/odoo --shell /usr/sbin/nologin odoo
```

Usuario de sistema `odoo`, con grupo propio, sin shell para entrar, y con `/var/lib/odoo` como directorio, que será el de los datos.

```
sudo -u postgres createuser --createdb odoo
```

Rol `odoo` en PostgreSQL, solo con permiso para crear bases. Se llama igual que el usuario de Linux para que funcione `peer`.

```
sudo -u odoo psql -l
```

Comprobación: lista las bases sin pedir contraseña.

## 2. Código de Odoo y entorno Python

**Dónde:** VPS.

⚠️ Si `git` no está instalado: `sudo apt install git`.

```
sudo mkdir /opt/odoo
sudo git clone --depth 1 --branch 18.0 https://github.com/odoo/odoo.git /opt/odoo/src
```

`/opt` es para programas instalados a mano. Se clona directamente en `src` para evitar rutas como `/opt/odoo/odoo/odoo`. `--depth 1` descarga solo el último commit; con `sudo`, el código es de root y Odoo no puede modificarlo.

```
sudo git -C /opt/odoo/src log -1 --oneline
```

Anotar el commit desplegado.

```
curl -LsSf https://astral.sh/uv/install.sh | sudo env UV_INSTALL_DIR=/usr/local/bin UV_NO_MODIFY_PATH=1 sh
```

Instala `uv` para todo el sistema en `/usr/local/bin`. Sin esas variables, con `sudo` acabaría en `/root/.local/bin`. (Para leer el instalador antes: cambiar `| sudo env … sh` por `| less`.)

```
sudo uv venv --python /usr/bin/python3 /opt/odoo/venv
```

Entorno virtual con el Python del sistema (3.12), de root.

Paquetes de compilación que necesitan algunas librerías (`psycopg2`, `python-ldap`):

```
sudo apt install gcc-x86-64-linux-gnu binutils libpython3.12-dev libldap-dev libpq-dev
```

| Paquete | Qué aporta |
|---|---|
| `gcc-x86-64-linux-gnu` | El compilador de C |
| `binutils` | El ensamblador (`as`) y el enlazador |
| `libpython3.12-dev` | `Python.h`, cabecera de Python |
| `libldap-dev` | `lber.h`, cabecera de LDAP |
| `libpq-dev` | `libpq-fe.h`, cabecera de PostgreSQL |

Si la instalación de librerías falla por otro fichero que falta, buscar qué paquete lo trae:

```
sudo apt install apt-file
sudo apt-file update
apt-file search --regexp '/NOMBRE_DEL_FICHERO$'
```

El nombre del fichero sale del mensaje de error. `apt-file update` descarga el índice de ficheros; sin él, `search` no encuentra nada.

```
sudo uv pip install --python /opt/odoo/venv/bin/python -r /opt/odoo/src/requirements.txt
```

Instala las librerías de Odoo dentro de su entorno.

```
/opt/odoo/venv/bin/python /opt/odoo/src/odoo-bin --version
```

Comprobación: `Odoo Server 18.0`.

*Opcional, comodidad de la shell:* `echo 'uvx --generate-shell-completion fish | source' > ~/.config/fish/completions/uvx.fish` (autocompletado de `uvx` en fish).

## 3. `odoo.conf`

**Dónde:** VPS.

```
sudo mkdir /etc/odoo
sudo touch /etc/odoo/odoo.conf
sudo chown root:odoo /etc/odoo/odoo.conf
sudo chmod 640 /etc/odoo/odoo.conf
```

`/etc` es la configuración. El fichero se crea vacío y se protege **antes** de escribir el secreto: root escribe, el grupo `odoo` lee, los demás nada (`rw-r-----`).

Hash de la contraseña maestra, para no guardarla en claro en el fichero:

```
/opt/odoo/venv/bin/python -c "from getpass import getpass; from passlib.context import CryptContext; print(CryptContext(schemes=['pbkdf2_sha512'], pbkdf2_sha512__rounds=600000).hash(getpass()))"
```

Odoo 18 acepta en `admin_passwd` un hash `pbkdf2_sha512` (600 000 rondas) o texto plano, obsoleto (`odoo/tools/config.py`, `crypt_context`). El gestor de bases solo escribe el hash si Odoo puede escribir en `odoo.conf`, y aquí no puede (es de root); además, se cierra en el paso 4 (`list_db = False`). Así que se calcula a mano con la `passlib` del venv. `getpass()` pide la contraseña sin mostrarla y sin dejarla en el historial. Sin `sudo`: solo se lee el venv. La salida empieza por `$pbkdf2-sha512$600000$`.

```
sudoedit /etc/odoo/odoo.conf
```

Contenido inicial (las opciones de seguridad se añaden en el paso 4, después de crear la base):

```ini
[options]
http_interface = 127.0.0.1
addons_path = /opt/odoo/src/addons,/opt/odoo/src/odoo/addons
data_dir = /var/lib/odoo
admin_passwd = HASH_CONTRASEÑA_MAESTRA
```

| Línea | Por qué |
|---|---|
| `[options]` | Sección obligatoria, en minúsculas |
| `http_interface` | Solo local: a Odoo se llega a través de nginx o del túnel, nunca directamente |
| `addons_path` | Dónde están los módulos |
| `data_dir` | *Filestore* y sesiones en `/var/lib/odoo` |
| `admin_passwd` | Hash de la contraseña maestra del gestor de bases. Sin la línea, el defecto es `admin`. El hash va tal cual, sin comillas (`$` no es especial en este fichero) |

```
sudo -u odoo cat /etc/odoo/odoo.conf
cat /etc/odoo/odoo.conf
```

Comprobación: el primero muestra el contenido; el segundo, como tu usuario, `Permission denied`.

```
sudo /opt/odoo/venv/bin/python -c "from getpass import getpass; from configparser import RawConfigParser; from passlib.context import CryptContext; c = RawConfigParser(); c.read('/etc/odoo/odoo.conf'); print(CryptContext(schemes=['pbkdf2_sha512']).verify(getpass(), c['options']['admin_passwd']))"
```

Comprobación: con la contraseña maestra del gestor, imprime `True`. Con el gestor cerrado no se usará hasta una restauración, así que conviene saber ahora que el hash quedó bien pegado.

⚠️ Sin probar en una instalación nueva: crear la base del paso 4 desde el gestor con `admin_passwd` ya en hash desde el principio (el código lo admite: `verify_admin_password`). En la máquina original se pasó a hash después.

## 4. Primer arranque y primera base

**Terminal 1 (VPS):**

```
sudo -u odoo /opt/odoo/venv/bin/python /opt/odoo/src/odoo-bin -c /etc/odoo/odoo.conf
```

Arranque a mano para probar. El log debe mostrar `Using configuration file at /etc/odoo/odoo.conf`.

**Terminal 2 (portátil):**

```
ssh -N -L 10000:127.0.0.1:8069 vps
```

Túnel: el puerto 10000 del portátil lleva al 8069 del VPS sin abrirlo a internet. `-N` = solo túnel, sin shell. (Se usa el 10000 porque el 8069 del portátil está ocupado.)

**Navegador del portátil:** `http://localhost:10000` → crear la base `test-db` desde el gestor, con la contraseña maestra, sin datos de demostración.

Cuando la base existe, cerrar el arranque a mano (`Ctrl+C` en la terminal 1) y añadir a `odoo.conf` (`sudoedit /etc/odoo/odoo.conf`):

```ini
list_db = False
dbfilter = ^test-db$
proxy_mode = True
```

| Línea | Por qué |
|---|---|
| `list_db = False` | Cierra el gestor de bases antes de exponer Odoo. La página sigue dando `200`, con el aviso «disabled by the administrator» |
| `dbfilter = ^test-db$` | Odoo solo usa esa base y no muestra selector |
| `proxy_mode = True` | Odoo se fía de las cabeceras `X-Forwarded-*` que pondrá nginx (IP, nombre y esquema reales) |

**Orden importante:** con `list_db = False` ya no se pueden crear bases desde el navegador. Por eso la base se crea antes.

## 5. Odoo como servicio

**Dónde:** VPS.

```
sudo nano /etc/systemd/system/odoo.service
```

```ini
[Unit]
Description=Odoo service
After=postgresql@16-main.service
Wants=postgresql@16-main.service

[Service]
User=odoo
ExecStart=/opt/odoo/venv/bin/python /opt/odoo/src/odoo-bin -c /etc/odoo/odoo.conf
Type=exec
UMask=0027
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

| Directiva | Por qué |
|---|---|
| `After=` / `Wants=` | Arranca después de PostgreSQL. `Wants=` y no `Requires=`: si PostgreSQL se para, Odoo sigue vivo y se reconecta solo |
| `User=odoo` | Sin privilegios; el grupo es el del usuario |
| `Type=exec` | systemd da el arranque por bueno cuando el programa se ha lanzado; si la ruta está mal, falla al momento |
| `UMask=0027` | Lo que cree Odoo nace sin escritura para el grupo y sin acceso para los demás |
| `Restart=on-failure` | Si se cae con error, systemd lo vuelve a lanzar |
| `WantedBy=multi-user.target` | Arranca al iniciar la máquina (`systemctl get-default` = `multi-user.target`) |

```
sudo systemctl daemon-reload
sudo systemd-analyze verify /etc/systemd/system/odoo.service
sudo systemctl enable odoo
sudo systemctl start odoo
systemctl status odoo
```

`daemon-reload`: systemd relee las units del disco. `verify`: sin salida = sin errores. `enable`: arranca en cada inicio. `start`: arranca ahora. `status`: `enabled` y `active (running)`.

```
sudo chmod -R g-w,o-rwx /var/lib/odoo
```

Ajusta los permisos de lo que se creó en el paso 4, antes de que existiera `UMask=`.

```
sudo ss -tlnp
```

Comprobación: Odoo (proceso `python`) solo en `127.0.0.1:8069`.

## 6. Informes PDF

**Dónde:** VPS.

```
cd ~
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.jammy_amd64.deb
sudo apt install ./wkhtmltox_0.12.6.1-3.jammy_amd64.deb
wkhtmltopdf -V
```

Build oficial con Qt parcheado: el `wkhtmltopdf` de Ubuntu existe, pero sin Qt parcheado y los PDF salen sin cabecera ni pie. El `./` hace que `apt` instale el fichero local. Comprobación: `0.12.6.1 (with patched qt)`. Queda **fuera** de las actualizaciones de `apt`.

```
sudo systemctl restart odoo
```

Para que Odoo detecte el programa.

**Navegador:** Ajustes → Técnico → Parámetros del sistema → crear `report.url` = `http://127.0.0.1:8069`. wkhtmltopdf pide los estilos, el logo y las fuentes desde el propio VPS; sin este parámetro usaría `web.base.url`, que puede no ser alcanzable desde el VPS. El nombre es exactamente `report.url`.

Al instalar la aplicación de Ventas, el módulo `account_peppol` pide la librería `phonenumbers`:

```
sudo uv pip install --python /opt/odoo/venv/bin/python phonenumbers
sudo systemctl restart odoo
```

Comprobación: un presupuesto en PDF sale con logo, cabecera, estilos y pie «Página 1 / 1».

## 7. nginx delante de Odoo (HTTP)

**Dónde:** VPS.

```
sudo apt install nginx
sudo rm /etc/nginx/sites-enabled/default
```

nginx arranca solo con el sitio `default` (página de bienvenida). Se desactiva borrando su enlace; el fichero de `sites-available` queda intacto.

```
sudoedit /etc/nginx/sites-available/odoo
```

```nginx
server {
        server_name odoo.mlozano.dev;
        listen 80;
        listen [::]:80;
        location / {
                proxy_pass http://127.0.0.1:8069;
                proxy_set_header X-Forwarded-Host $http_host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
}
```

`proxy_pass` a `127.0.0.1` (no `localhost`, que también resuelve a `::1`). Las tres cabeceras pasan a Odoo el nombre, la IP y el esquema del visitante. Revisar el `;` de cada línea: un olvido puede pasar `nginx -t`.

```
sudo ln -s /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/odoo
sudo nginx -t
sudo systemctl reload nginx
```

`ln -s`: activa el sitio (primero el destino, después el enlace). `nginx -t`: comprueba la configuración; **siempre** antes de recargar. `reload`: aplica sin cortar conexiones.

## 8. DNS

**Dónde:** panel de Cloudflare (navegador) y portátil.

En la zona `mlozano.dev`, registros para `odoo`:

| Tipo | Nombre | Contenido | Proxy |
|---|---|---|---|
| `A` | `odoo` | IPv4 del VPS | **DNS only** (nube gris) |
| `AAAA` | `odoo` | IPv6 del VPS | **DNS only** |

DNS only para que el único proxy sea nginx.

```
dig +short odoo.mlozano.dev
```

**Portátil.** Comprobación: devuelve la IPv4 del VPS. Para ver el dato en la fuente, sin cachés: `dig @autumn.ns.cloudflare.com odoo.mlozano.dev A` (flag `aa`).

```
curl -4 -sI http://odoo.mlozano.dev/web
```

**Portátil.** Comprobación: `303` con `Location` relativa, y en el VPS `sudo journalctl -u odoo -n 5 --no-pager` muestra tu IP pública (no `127.0.0.1`). El navegador todavía no abre la página: `.dev` exige HTTPS.

## 9. HTTPS con certbot

**Dónde:** VPS, salvo lo indicado.

```
cp /etc/nginx/sites-available/odoo ~/odoo.nginx.antes-certbot
```

Copia del sitio para ver después qué cambia certbot. En la home, no dentro de `/etc/nginx`.

```
sudo apt install certbot python3-certbot-nginx
```

Desde los repositorios de Ubuntu, sin snap. El plugin de nginx es el paquete `python3-certbot-nginx`.

```
sudo certbot --nginx -d odoo.mlozano.dev
```

Pide el certificado a Let's Encrypt (que valida el dominio por el puerto 80) y edita el sitio: el bloque original pasa al 443 y aparece un bloque nuevo en el 80 que redirige a `https://`. La primera vez pide correo y aceptar los términos.

```
diff -u ~/odoo.nginx.antes-certbot /etc/nginx/sites-available/odoo
```

Qué ha añadido certbot (`+`) y quitado (`-`).

```
sudo certbot renew --dry-run
```

Simulacro de renovación: debe terminar con `all simulated renewals succeeded`. La renovación real la lanza `certbot.timer` dos veces al día (`systemctl list-timers`).

**Websocket** (notificaciones en vivo). Copia previa y edición del sitio:

```
cp /etc/nginx/sites-available/odoo ~/odoo.nginx.antes-websocket
sudoedit /etc/nginx/sites-available/odoo
```

Al principio del fichero, antes del primer `server {` y **sin** `http {` alrededor:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

Define `$connection_upgrade`, que nginx no trae. El sitio ya está dentro de `http` por el `include` de `nginx.conf`.

Dentro del bloque del 443, junto a `location /`:

```nginx
        location /websocket {
                proxy_pass http://127.0.0.1:8069;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
                proxy_set_header X-Forwarded-Host $http_host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
```

nginx no reenvía `Upgrade` ni `Connection` si no se le dice. `proxy_http_version 1.1` hace falta en nginx anterior a 1.29.7 (comprobar con `nginx -v`). Las `X-Forwarded-*` se repiten porque no se heredan de `location /`.

```
sudo nginx -t
sudo systemctl reload nginx
```

**Comprobaciones:**

| Dónde | Qué | Resultado esperado |
|---|---|---|
| Portátil | `curl -I http://odoo.mlozano.dev/web/login` | `301` con `Location: https://odoo.mlozano.dev/web/login` |
| Portátil, navegador | `https://odoo.mlozano.dev` | Carga sin aviso; certificado de Let's Encrypt |
| VPS | `ls -l /etc/letsencrypt/live/odoo.mlozano.dev/` sin `sudo` | Error de permisos: tu usuario no lee la clave |
| Navegador → VPS | Entrar como administrador y luego `sudo -u odoo psql -d test-db -c "SELECT value FROM ir_config_parameter WHERE key = 'web.base.url';"` | `https://odoo.mlozano.dev` |
| Navegador | Imprimir un presupuesto | PDF con cabecera y pie |
| VPS | Cerrar las pestañas de Odoo y `sudo grep 'GET /websocket' /var/log/nginx/access.log \| tail -n 3` | Código `101` |

**HSTS y cookie de sesión segura.** certbot no añade ninguna de las dos. Copia previa (se borra tras comprobar) y edición:

```
cp /etc/nginx/sites-available/odoo ~/odoo.nginx.antes-hsts
sudoedit /etc/nginx/sites-available/odoo
```

Dentro del bloque del 443, **fuera** de los `location` (por ejemplo, debajo de `server_name`):

```nginx
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
        proxy_cookie_flags session_id samesite=lax secure;
```

| Línea | Por qué |
|---|---|
| `add_header Strict-Transport-Security …` | El navegador usará solo HTTPS con este nombre durante un año. En el bloque del 80 no sirve: el navegador ignora HSTS recibida por `http` |
| `proxy_cookie_flags session_id samesite=lax secure` | Odoo crea `session_id` solo con `HttpOnly`; nginx le añade `Secure` (solo viaja por HTTPS) y `SameSite=Lax` (protección contra CSRF). Requiere nginx ≥ 1.19.3 |
| A nivel de `server` | Los dos `location` heredan las dos directivas. Si algún `location` define su propio `add_header` o `proxy_cookie_flags`, deja de heredar **todos** los del nivel superior |

```
sudo nginx -t
sudo systemctl reload nginx
```

**Comprobaciones:**

| Dónde | Qué | Resultado esperado |
|---|---|---|
| Portátil | `curl -I http://odoo.mlozano.dev/` | `301` a `https://` y **sin** `Strict-Transport-Security` |
| Portátil | `curl -I https://odoo.mlozano.dev/` | `Strict-Transport-Security: max-age=31536000; includeSubDomains` y `Set-Cookie: session_id=…; HttpOnly; Path=/; Secure; SameSite=Lax` |
| Navegador | Cerrar sesión, entrar de nuevo y navegar | La sesión se mantiene |

**Contraseña del administrador de la base.** Si en el paso 7 se inició sesión en Odoo por `http`, esa contraseña viajó en claro: cambiarla ahora desde la interfaz (avatar → *Mi perfil* → *Seguridad de la cuenta* → *Cambiar contraseña*), con la nueva guardada antes en el gestor. Para no tener que hacerlo, no iniciar sesión por `http` en el paso 7.

## 10. Pendiente de completar

- Contrastar esta guía con el journal de `sudo` del VPS (Sesión 11).

## Secretos

| Secreto | Dónde vive | En este repo |
|---|---|---|
| Contraseña maestra | En claro, solo en el gestor de contraseñas. En `/etc/odoo/odoo.conf` (`root:odoo 640`), solo su hash (`admin_passwd`) | **No** |
| Contraseña del administrador de `test-db` | Hash en la tabla `res_users` de la base; en claro, solo en el gestor de contraseñas | **No** |
| Clave privada del certificado | `/etc/letsencrypt/live/odoo.mlozano.dev/privkey.pem` (solo root) | **No** |
