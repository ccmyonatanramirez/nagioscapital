# Instalación y configuración de NRPE en el cliente Linux (compilado desde código fuente)

Este documento describe el proceso real usado para dejar listo el **lado cliente** de NRPE en un host Linux (Ubuntu/Debian o CentOS/RHEL): compilación desde código fuente de los Nagios Plugins y de NRPE, creación del usuario de servicio, configuración de `nrpe.cfg`, y puesta en marcha como servicio systemd.

Los paquetes fuente utilizados están alojados en el repositorio interno:

- [`nagios-plugins-2.5.tar.gz`](https://github.com/ccmyonatanramirez/nagioscapital/blob/main/nagios-plugins-2.5.tar.gz)
- [`nrpe-4.1.3.tar.gz`](https://github.com/ccmyonatanramirez/nagioscapital/blob/main/nrpe-4.1.3.tar.gz)

## Índice

1. [Requisitos previos](#requisitos-previos)
2. [Instalar dependencias de compilación](#instalar-dependencias-de-compilación)
3. [Crear el usuario `nagios`](#crear-el-usuario-nagios)
4. [Compilar e instalar los Nagios Plugins](#compilar-e-instalar-los-nagios-plugins)
5. [Compilar e instalar NRPE](#compilar-e-instalar-nrpe)
6. [Configurar `nrpe.cfg`](#configurar-nrpecfg)
7. [Crear el servicio y arrancarlo](#crear-el-servicio-y-arrancarlo)
8. [Firewall y SELinux](#firewall-y-selinux)
9. [Verificación local](#verificación-local)
10. [Solución de problemas comunes](#solución-de-problemas-comunes)

---

## Requisitos previos

- Acceso root o sudo en el host a monitorear.
- Conectividad de red desde el servidor Nagios hacia este host por el puerto **5666/tcp**.
- Tener a mano la IP del servidor Nagios central, para autorizarla en `allowed_hosts`.

---

## Instalar dependencias de compilación

**CentOS / RHEL:**

```bash
sudo yum install -y gcc glibc-common openssl openssl-devel perl wget make automake autoconf
```

**Ubuntu / Debian** (paquetes equivalentes):

```bash
sudo apt update
sudo apt install -y gcc build-essential openssl libssl-dev perl wget make automake autoconf
```

---

## Crear el usuario `nagios`

NRPE debe correr bajo un usuario y grupo dedicados, no como root:

```bash
sudo groupadd nagios
sudo useradd -g nagios -r -s /sbin/nologin nagios
```

---

## Compilar e instalar los Nagios Plugins

Se compilan **primero** los plugins, ya que NRPE depende de que existan para poder ejecutarlos.

```bash
cd /tmp
wget https://github.com/ccmyonatanramirez/nagioscapital/raw/main/nagios-plugins-2.5.tar.gz
tar -xzf nagios-plugins-2.5.tar.gz
cd nagios-plugins-2.5

./configure
make
sudo make install
```

Por defecto quedan instalados en `/usr/local/nagios/libexec/`.

---

## Compilar e instalar NRPE

```bash
cd /tmp
wget https://github.com/ccmyonatanramirez/nagioscapital/raw/main/nrpe-4.1.3.tar.gz
tar -xzf nrpe-4.1.3.tar.gz
cd nrpe-4.1.3

./configure
make all
sudo make install
sudo make install-config
sudo make install-init
```

- `make install` instala el binario `nrpe` (y `check_nrpe`) en `/usr/local/nagios/bin/` y `/usr/local/nagios/libexec/`.
- `make install-config` genera el archivo de configuración por defecto en `/usr/local/nagios/etc/nrpe.cfg`.
- `make install-init` instala el servicio systemd (`nrpe.service`).

---

## Configurar `nrpe.cfg`

```bash
sudo nano /usr/local/nagios/etc/nrpe.cfg
```

Ajuste los parámetros clave:

```ini
server_address=IP_DEL_HOST_LOCAL
allowed_hosts=127.0.0.1,IP_DEL_SERVIDOR_NAGIOS
dont_blame_nrpe=0
```

Y agregue los comandos personalizados que va a exponer, por ejemplo:

```ini
command[check_disk]=/usr/local/nagios/libexec/check_disk -w 20% -c 10% -p /
command[check_load]=/usr/local/nagios/libexec/check_load -w 5,4,3 -c 10,8,6
command[check_users]=/usr/local/nagios/libexec/check_users -w 5 -c 10
command[check_procs]=/usr/local/nagios/libexec/check_procs -w 200 -c 250
command[check_mem]=/usr/local/nagios/libexec/check_mem.sh -w 80 -c 90
```

---

## Crear el servicio y arrancarlo

```bash
sudo systemctl daemon-reload
sudo systemctl enable nrpe
sudo systemctl start nrpe
sudo systemctl status nrpe
```

---

## Firewall y SELinux

**Firewall (`firewalld`, CentOS):**

```bash
sudo firewall-cmd --permanent --add-port=5666/tcp
sudo firewall-cmd --reload
```

**Firewall (`ufw`, Ubuntu):**

```bash
sudo ufw allow from IP_DEL_SERVIDOR_NAGIOS to any port 5666 proto tcp
sudo ufw reload
```

**SELinux (CentOS, si está en `enforcing`):**

```bash
sestatus
sudo semanage port -a -t nrpe_port_t -p tcp 5666
```

(Si el paquete `policycoreutils-python-utils` no está instalado, `semanage` no estará disponible — instálelo con `sudo yum install -y policycoreutils-python-utils`)

---

## Verificación local

```bash
/usr/local/nagios/libexec/check_nrpe -H localhost
```

Debe responder con la versión de NRPE instalada, por ejemplo:

```
NRPE v4.1.3
```

Pruebe también un comando personalizado:

```bash
/usr/local/nagios/libexec/check_nrpe -H localhost -c check_load
```

Con esto confirmado, el host queda listo del lado cliente — solo falta que desde el servidor Nagios se agregue la definición de host/servicio correspondiente.

---

## Solución de problemas comunes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `./configure` falla por falta de OpenSSL | Faltan las librerías de desarrollo | Confirmar instalación de `openssl-devel` (CentOS) / `libssl-dev` (Ubuntu) |
| `make install-init` no crea el servicio | Versión de NRPE sin soporte systemd, o target no soportado en esa distro | Revisar manualmente `/etc/systemd/system/nrpe.service` tras la instalación, o crear la unidad a mano si no se generó |
| `Connection refused` al probar `check_nrpe` | El servicio no está corriendo | `systemctl status nrpe`, revisar `journalctl -xeu nrpe` |
| `Connection timed out` | Firewall bloqueando el puerto 5666 | Revisar reglas de `firewalld`/`ufw` y de cualquier firewall de red intermedio |
| SELinux bloquea la conexión (CentOS) | Puerto no registrado en el contexto `nrpe_port_t` | `semanage port -a -t nrpe_port_t -p tcp 5666` |
| Permisos denegados al ejecutar plugins | El usuario `nagios` no tiene permisos sobre algún recurso que consulta el plugin (ej. discos, procesos) | Revisar permisos del recurso o agregar el usuario `nagios` al grupo correspondiente |

---

## Resumen rápido del proceso completo

```bash
# Dependencias
sudo yum install -y gcc glibc-common openssl openssl-devel perl wget make automake autoconf

# Usuario de servicio
sudo groupadd nagios
sudo useradd -g nagios -r -s /sbin/nologin nagios

# Plugins
cd /tmp && wget https://github.com/ccmyonatanramirez/nagioscapital/raw/main/nagios-plugins-2.5.tar.gz
tar -xzf nagios-plugins-2.5.tar.gz && cd nagios-plugins-2.5
./configure && make && sudo make install

# NRPE
cd /tmp && wget https://github.com/ccmyonatanramirez/nagioscapital/raw/main/nrpe-4.1.3.tar.gz
tar -xzf nrpe-4.1.3.tar.gz && cd nrpe-4.1.3
./configure && make all && sudo make install && sudo make install-config && sudo make install-init

# Servicio
sudo systemctl daemon-reload && sudo systemctl enable nrpe && sudo systemctl start nrpe
```
