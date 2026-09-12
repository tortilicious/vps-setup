# vps-setup

Configuración y procedimiento de despliegue del VPS: Ubuntu, systemd, nginx, PostgreSQL, Odoo.

Este documento es un **runbook**: está escrito para reproducir la instalación desde cero en una máquina nueva. Los pasos son imperativos y verificables.

**Estado actual:** Bloque 0 completado (servidor operativo, acceso SSH solo por clave).

---

## 1. La máquina

| | |
|---|---|
| Proveedor | Netcup |
| Plan | VPS 500 G12 |
| Hardware | 2 vCore · 4 GB DDR5 ECC · 128 GB NVMe |
| Ubicación | Núremberg (DE) |
| Coste | ~7 €/mes, sin permanencia |
| SO | Ubuntu 24.04 LTS (imagen minimal, UEFI amd64) |
| Hostname | `backend-lab` |
| Panel | SCP de Netcup |

Se consultaron otros proveedores; quedaron descartados por relación hardware/precio.

---

## 2. Decisiones y por qué

### Sistema operativo: Ubuntu

Se elige Ubuntu por motivos prácticos: es lo que se encuentra con más frecuencia en despliegues reales de Odoo. Debian habría sido la opción más didáctica —un Linux más limpio, con menos capas propias de distribución— pero se descarta porque el objetivo es trabajar sobre lo que hay en producción.

### Versión: 24.04 LTS

Tres criterios, en este orden:

1. **Rodaje.** 24.04 salió en abril de 2024: lleva más de dos años en servicio, con los problemas habituales ya documentados y resueltos por la comunidad.
2. **Ventana de soporte.** Las LTS tienen cinco años de soporte estándar. 24.04 llega hasta abril de 2029, con margen de sobra para la duración del proyecto.
3. **Adopción real.** 26.04, aunque más moderna, es demasiado reciente: menos información disponible para resolver incidencias y no es lo que hay hoy en los servidores de clientes.

**22.04 se descarta** pese a ser la más rodada: su soporte estándar termina en abril de 2027, en mitad del proyecto.

### Imagen minimal

No es una elección: todas las imágenes oficiales que ofrece Netcup son minimal. La consecuencia operativa es que el sistema viene sin herramientas de conveniencia y hay que instalarlas a mano según se necesiten (el primer caso fue `nano`).

### Usuario propio con sudo, en lugar de root

- **Menos exposición.** Los privilegios de administración se usan solo cuando hacen falta, no de forma permanente. Un comando mal escrito como root es irreversible.
- **Trazabilidad.** `sudo` registra quién ejecutó qué. Con acceso directo de root, todas las acciones son anónimas.
- **Contención.** Si un proceso del usuario se ve comprometido, hereda los permisos del usuario, no los de root.

Pertenecer al grupo `sudo` no concede privilegios permanentes: concede el derecho a solicitarlos, comando a comando, autenticándose con la contraseña del propio usuario.

### Clave SSH en lugar de contraseña

Un servidor con IP pública recibe intentos de acceso automatizados desde el primer día. Mientras el servidor acepte contraseñas, esos bots tienen un método al que llamar. Con autenticación por clave, el método deja de existir: no hay dónde probar contraseñas.

**Mecanismo, en dos fases:**

**Fase 1 — autenticación del servidor.** Al conectar, el servidor presenta su clave de host. El cliente muestra su huella (fingerprint) y pide confirmación. Esa huella se coteja contra la que publica el SCP de Netcup: sirve para verificar que se está hablando con el servidor real y no con un intermediario. Una vez aceptada, el cliente la guarda en `~/.ssh/known_hosts` y la verifica sola en conexiones posteriores. Si la huella cambia, SSH avisa de un posible ataque man-in-the-middle.

**Fase 2 — autenticación del usuario.** Tras negociar la clave de sesión, el servidor envía un **dato aleatorio** (un reto, distinto en cada conexión). El cliente lo firma con su clave privada y devuelve la firma. El servidor la verifica contra la clave pública registrada en `~/.ssh/authorized_keys` de ese usuario. Si cuadra, concede acceso.

**La clave privada nunca sale de la máquina local.** No viaja por la red, ni siquiera cifrada, así que no hay nada que interceptar. Y como el reto es irrepetible, una captura de tráfico no permite reutilizar nada.

### Endurecimiento de SSH: orden de las dos directivas

| Directiva | Efecto |
|---|---|
| `PermitRootLogin no` | Root no puede iniciar sesión por SSH |
| `PasswordAuthentication no` | El servidor deja de ofrecer autenticación por contraseña |

**El orden importa.** `PasswordAuthentication no` se aplica **después** de haber verificado que el acceso por clave funciona. Invertir el orden deja la máquina inaccesible por SSH.

---

## 3. Procedimiento

### 3.0 — Antes de empezar

Localizar la consola VNC en el SCP y comprobar que se puede acceder por ella (ver §5). Es la vía de recuperación si algo sale mal, y hay que verificarla cuando no se necesita.

### 3.1 — Instalar la imagen

Desde el SCP: *Media → Images*, elegir `Ubuntu 24.04.x UEFI amd64`. Al terminar, el panel entrega la contraseña inicial de root y las huellas de las claves de host.

Guardar en el gestor de contraseñas: IP, contraseña de root, credenciales del SCP y las huellas.

### 3.2 — Primer acceso como root

```bash
ssh root@<IP>
```

En la primera conexión, SSH muestra la huella del servidor y pide confirmación. **Cotejarla con la del SCP antes de aceptar** (el servidor publica varias, una por tipo de clave; la que se muestra es la del algoritmo negociado, normalmente ED25519).

Comprobar dónde se está y qué versión hay:

```bash
pwd
hostname
lsb_release -a          # o: cat /etc/os-release
```

### 3.3 — Crear el usuario y darle sudo

```bash
adduser <usuario>              # interactivo: crea home, shell y pide contraseña
usermod -aG sudo <usuario>     # -a añade, -G grupo suplementario
groups <usuario>               # verificar
```

> `usermod -G` **sin** `-a` reemplaza todos los grupos del usuario en vez de añadir. Usar siempre ambas opciones juntas.

### 3.4 — Generar el par de claves (en la máquina local)

```bash
ssh-keygen -t ed25519 -C "<comentario identificativo>"
```

Se ejecuta en el **portátil**, no en el servidor. Genera dos ficheros en `~/.ssh/`:

- `id_ed25519` — privada. No se copia, no se comparte, no se sube a ningún sitio.
- `id_ed25519.pub` — pública. Es la que va al servidor.

Conviene poner *passphrase*: cifra la clave privada en disco, de modo que un fichero robado sin la frase no sirve.

### 3.5 — Instalar la clave pública en el servidor

```bash
ssh-copy-id <usuario>@<IP>
```

Añade la clave pública como una línea en `~/.ssh/authorized_keys` del usuario en el servidor.

Permisos requeridos (SSH ignora la clave en silencio si son más laxos):

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 3.6 — Alias de conexión (máquina local)

En `~/.ssh/config`:

```
Host vps
    HostName <IP>
    User <usuario>
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

A partir de aquí, `ssh vps`. Este fichero lo usan también `scp`, `rsync` y Git.

### 3.7 — Instalar herramientas mínimas

```bash
sudo apt update
sudo apt install nano
```

### 3.8 — Endurecer SSH

Editar `/etc/ssh/sshd_config` (el fichero del **daemon**; `ssh_config`, sin la `d`, es el del cliente y no sirve aquí):

```bash
sudo nano /etc/ssh/sshd_config
```

Dejar estas tres directivas activas (sin `#` delante) y cada una en su línea:

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
```

Comprobar cómo quedaron:

```bash
grep -n "^PermitRootLogin\|^PasswordAuthentication\|^KbdInteractive" /etc/ssh/sshd_config
```

> **`KbdInteractiveAuthentication`**: si se deja en `yes`, PAM puede seguir ofreciendo un camino a la autenticación por contraseña aunque `PasswordAuthentication` esté en `no`.

> **Directorio `Include`**: `sshd_config` incluye `/etc/ssh/sshd_config.d/`. Los fragmentos de ahí se leen **antes** y en OpenSSH **gana el primer valor leído**. Si el cambio no surte efecto, revisar ese directorio. En esta instalación estaba vacío.

> **Bloques `Match`**: todo lo que va después de un `Match` activo se aplica solo a esa condición. Colocar las directivas globales **antes** de cualquier `Match`.

### 3.9 — Validar y aplicar

**No cerrar la sesión actual en ningún momento.**

```bash
sudo sshd -t                    # valida la sintaxis sin aplicar. Silencio = correcto
sudo systemctl restart ssh
systemctl status ssh            # confirmar que la hora de arranque es la de ahora
```

> `sshd -t` **sin `sudo`** falla con `no hostkeys available -- exiting`. Es un error de permisos (las claves privadas de host solo las lee root), no de configuración.

> `systemctl reload ssh` **no bastó** para que la nueva lista de métodos de autenticación surtiera efecto; hizo falta `restart`. Reiniciar el servicio no corta la sesión activa: cada sesión cuelga de su propio proceso hijo, independiente del proceso que escucha.

---

## 4. Verificación

Desde una **segunda terminal**, sin cerrar la primera:

```bash
ssh -v vps
```

Comprobar en la salida:

| Línea | Valor esperado |
|---|---|
| `Server host key: ssh-ed25519 SHA256:...` | Coincide con la huella guardada del SCP |
| `Host '<IP>' is known and matches the ED25519 host key` | El host está verificado en `known_hosts` |
| `Authentications that can continue:` | **`publickey`** y nada más |
| `Authenticated to <IP> ... using "publickey"` | Autenticado por clave, no por contraseña |

Prueba adicional: forzar autenticación por contraseña y confirmar que el servidor la rechaza.

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no vps
```

---

## 5. Recuperación

Si se pierde el acceso por SSH (clave privada perdida, `sshd_config` roto, servicio caído), **no hace falta reinstalar el sistema**.

**Consola VNC:** SCP → servidor → *Screen*.

Es un monitor y un teclado conectados directamente a la máquina: no pasa por SSH ni depende de que `sshd` funcione. Se entra con usuario y contraseña (root o el usuario propio, ambos válidos, porque esta vía no está sujeta a `sshd_config`).

Una vez dentro se puede reeditar `/etc/ssh/sshd_config`, añadir una clave pública nueva a `authorized_keys` o reiniciar el servicio.

**Dos avisos:**
- El teclado suele estar en distribución **US**: los símbolos aparecen en posiciones distintas.
- Copiar y pegar no funciona bien o no funciona. Hay que teclearlo todo.

---

## 6. Comandos de referencia

### Diagnóstico de SSH

```bash
ssh -v vps                                  # traza de la conexión
sudo sshd -t                                # validar sintaxis de sshd_config
sudo sshd -T                                # volcar la configuración efectiva
systemctl status ssh ssh.socket             # estado del servicio
sudo journalctl -u ssh -n 50                # últimos 50 eventos del servicio
sudo grep sshd /var/log/auth.log            # intentos de autenticación
```

### Sistema

```bash
pwd                                         # dónde estoy
hostname                                    # en qué máquina estoy
ls -la <ruta>                               # listado largo, incluidos ocultos
lsb_release -a                              # versión de Ubuntu
grep -n "<patrón>" <fichero>                # buscar con número de línea
sudo apt update && sudo apt install <paq>   # instalar paquetes
```

### Gestión de servicios

```bash
sudo systemctl restart <servicio>
sudo systemctl reload <servicio>            # releer config sin reiniciar (no siempre basta)
systemctl status <servicio>
```

### Terminfo remoto

Si el terminal local no está en la base de datos del servidor (`Error opening terminal: xterm-<nombre>`), instalar su definición en el servidor desde la máquina local:

```bash
infocmp -x xterm-ghostty | ssh vps -- tic -x -
```

Esto lo instala en `~/.terminfo` del usuario remoto, **no** a nivel de sistema: root seguirá sin encontrarlo y `sudo nano` seguirá fallando. Solución puntual:

```bash
sudo TERM=xterm-256color nano <fichero>
```

Solución permanente: activar la integración de shell en el emulador de terminal local (en Ghostty, `shell-integration-features = ssh-env,ssh-terminfo,sudo` en `~/.config/ghostty/config`).

---

## 7. Pendiente

Bloque 4 del roadmap — el servidor todavía no está endurecido más allá de SSH:

- [ ] `ufw` (cortafuegos)
- [ ] `fail2ban`
- [ ] Endurecimiento adicional de SSH
- [ ] Actualizaciones desatendidas
- [ ] Backup automatizado, **con restauración probada en máquina limpia**

Otros bloques:

- [ ] Bloque 1 — el sistema por dentro
- [ ] Bloque 2 — systemd
- [ ] Bloque 3 — Odoo 18 desplegado a mano
- [ ] Bloque 5 — romperlo y diagnosticar

---

## Convenciones

- `<IP>`, `<usuario>`, `<paq>` son marcadores: sustituir por el valor real.
- **Ningún secreto vive en este repositorio.** Contraseñas, claves privadas y credenciales están en el gestor de contraseñas.
