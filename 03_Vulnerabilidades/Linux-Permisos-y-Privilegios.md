# Linux — Permisos y Privilegios

## Descripción

Este documento explica cómo funciona Linux por dentro en cuanto a usuarios, permisos y privilegios. Es la base teórica necesaria para entender la escalada de privilegios, los ataques SUID y las Linux Capabilities.

---

## El modelo de seguridad de Linux

Linux es un sistema **multiusuario**. Desde el primer día fue diseñado para que varios usuarios usen la misma máquina simultáneamente sin que unos puedan meterse en los archivos de otros. Todo el sistema de seguridad gira alrededor de esta idea.

### Los tres tipos de usuario

**root (UID 0)** — El superusuario. Puede hacer absolutamente cualquier cosa en el sistema. No hay ninguna restricción que le aplique.

**Usuarios normales (UID 1000+)** — Cuentas de personas reales. Tienen acceso limitado a sus propios archivos y a lo que el sistema les permite explícitamente.

**Usuarios de sistema (UID 1-999)** — Cuentas que no pertenecen a personas sino a servicios: `www-data` para Apache, `mysql` para MySQL, `nobody` para procesos sin privilegios. Existen para que cada servicio corra con los mínimos permisos necesarios.

---

## El sistema de permisos — DAC

DAC (Discretionary Access Control) es el sistema de permisos estándar de Linux. Lo ves cuando haces `ls -la`:

```
-rwxr-xr-x 1 root  root  531928 tar
-rw-r--r-- 1 cyber cyber     48 user.txt
drwxr-xr-x 2 cyber cyber   4096 directorio/
```

### Anatomía de una línea

```
-rwxr-xr-x   1   root   root   531928   tar
│└─────────   │   │      │      │        └─ nombre del archivo
│             │   │      │      └─ tamaño en bytes
│             │   │      └─ grupo propietario
│             │   └─ usuario propietario
│             └─ número de enlaces
└─ tipo de archivo + permisos (10 caracteres)
```

### Tipo de archivo (primer carácter)

| Carácter | Tipo |
|----------|------|
| `-` | Archivo normal |
| `d` | Directorio |
| `l` | Enlace simbólico |
| `s` | Socket |
| `p` | Pipe |

### Los 9 caracteres de permisos

Se dividen en tres grupos de tres, cada uno para un conjunto de usuarios diferente:

```
rwx  r-x  r-x
│    │    └─ otros (todos los demás usuarios)
│    └─ grupo propietario
└─ usuario propietario
```

Cada grupo tiene tres bits:

| Carácter | Permiso | En archivo | En directorio |
|----------|---------|------------|---------------|
| `r` | read | Leer contenido | Listar archivos con ls |
| `w` | write | Modificar contenido | Crear/borrar archivos dentro |
| `x` | execute | Ejecutar como programa | Entrar con cd |
| `-` | sin permiso | denegado | denegado |

### Ejemplos reales

```
-rw-r--r-- 1 root root /etc/passwd
```
- root → lectura y escritura
- grupo root → solo lectura
- todos → solo lectura (legible por cualquier usuario)

```
-rw-r----- 1 root shadow /etc/shadow
```
- root → lectura y escritura
- grupo shadow → solo lectura
- todos los demás → **sin acceso** — por eso no puedes leerlo como usuario normal

```
-rwsr-xr-x 1 root root /usr/bin/passwd
```
- La `s` en lugar de `x` en el grupo del propietario → **bit SUID activo**

---

## Los archivos más importantes del sistema

### /etc/passwd — directorio de usuarios

Legible por todos los usuarios. Contiene información de cada cuenta pero **no las contraseñas reales**.

```
cyber:x:1000:1000::/home/cyber:/bin/bash
root:x:0:0:root:/root:/bin/bash
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

| Campo | Ejemplo | Significado |
|-------|---------|-------------|
| 1 | `cyber` | Nombre de usuario |
| 2 | `x` | La `x` indica que la contraseña está en /etc/shadow |
| 3 | `1000` | UID — identificador numérico del usuario |
| 4 | `1000` | GID — identificador del grupo principal |
| 5 | (vacío) | Comentario o descripción |
| 6 | `/home/cyber` | Directorio home |
| 7 | `/bin/bash` | Shell por defecto al hacer login |

> En pentesting: `/usr/sbin/nologin` o `/bin/false` como shell significa que la cuenta no puede hacer login interactivo — típico en cuentas de servicio.

### /etc/shadow — los hashes de contraseñas

Solo legible por root y el grupo `shadow`. Es el archivo más valioso en una escalada de privilegios.

```
root:$y$j9T$M3BDk...:18919:0:99999:7:::
cyber:$y$j9T$x6sD...:18919:0:99999:7:::
daemon:*:18919:0:99999:7:::
```

| Campo | Significado |
|-------|-------------|
| `root` | Nombre de usuario |
| `$y$j9T$...` | Hash de la contraseña |
| `18919` | Días desde 1/1/1970 en que se cambió la contraseña |
| `0` | Mínimo de días entre cambios de contraseña |
| `99999` | Máximo de días antes de que expire |
| `*` | Sin contraseña — la cuenta no puede hacer login |
| `!` | Cuenta bloqueada |

### Formato del hash

```
$y$j9T$M3BDdkxYOlVM6ECoqwUFs.$Wyz40CNLlZCFN6Xltv9AA...
│  │   │                       └─ hash resultado
│  │   └─ salt (valor aleatorio único por contraseña)
│  └─ parámetros del algoritmo
└─ identificador del algoritmo
```

| Prefijo | Algoritmo | Seguridad |
|---------|-----------|-----------|
| `$1$` | MD5 | Muy débil — crackeable rápido |
| `$5$` | SHA-256 | Medio |
| `$6$` | SHA-512 | Bueno |
| `$2y$` | bcrypt | Muy bueno |
| `$y$` | yescrypt | Excelente — el más moderno y lento |

> El salt hace que dos usuarios con la misma contraseña tengan hashes diferentes. Impide usar tablas precomputadas (rainbow tables).

### /etc/sudoers — quién puede usar sudo

Define qué usuarios pueden ejecutar qué comandos como root con `sudo`:

```
root    ALL=(ALL:ALL) ALL
cyber   ALL=(ALL) NOPASSWD: /usr/bin/vim
%admin  ALL=(ALL) ALL
```

- `NOPASSWD` → puede ejecutar ese comando como root sin introducir contraseña — **vector de escalada directo**
- `%admin` → el `%` indica un grupo, no un usuario

### /etc/crontab — tareas programadas

```
* * * * * root /opt/script.sh
17 * * * * root /usr/bin/backup.sh
```

Si hay un script ejecutándose como root periódicamente y tú tienes permisos de escritura sobre él, puedes inyectar comandos maliciosos que se ejecutarán como root.

### ~/.ssh/authorized_keys — claves SSH autorizadas

Contiene las claves públicas que pueden conectarse por SSH sin contraseña. Si puedes escribir en `/root/.ssh/authorized_keys`, añades tu clave pública y te conectas como root directamente.

### /proc — información del sistema en tiempo real

```
/proc/version        → versión exacta del kernel
/proc/net/netlink    → sockets de red activos
/proc/[PID]/         → información de cada proceso en ejecución
/proc/[PID]/environ  → variables de entorno del proceso (pueden tener credenciales)
```

---

## El bit SUID — primera forma de escalar privilegios

### El problema que resuelve

`/usr/bin/passwd` necesita escribir en `/etc/shadow` para cambiar tu contraseña. Pero `/etc/shadow` solo lo puede escribir root. ¿Cómo puede un usuario normal cambiar su propia contraseña sin ser root?

### La solución

El **bit SUID**. Cuando un ejecutable tiene SUID activado, se ejecuta con los permisos del **propietario del archivo** en lugar del usuario que lo lanza.

```
-rwsr-xr-x 1 root root /usr/bin/passwd
    └─ 's' en lugar de 'x' → SUID activado
```

Cuando `cyber` ejecuta `passwd`, el proceso corre con **EUID (UID efectivo) de root** aunque tú seas `cyber`. Eso le permite escribir en `/etc/shadow` puntualmente.

### UID real vs EUID efectivo

- **UID real** — quién eres tú (`cyber`, uid=1000)
- **EUID efectivo** — con qué permisos actúa el proceso en este momento

Con SUID en un binario de root: UID real = 1000 (cyber), EUID = 0 (root).

### Por qué es peligroso

Si un binario que permite ejecutar comandos arbitrarios tiene SUID y pertenece a root, cualquier usuario puede usarlo para obtener una shell de root:

```shell
# bash con SUID
bash -p          # -p mantiene el EUID efectivo (root)
whoami           # root

# find con SUID
find . -exec /bin/bash -p \; -quit

# vim con SUID
vim -c ':!/bin/bash'
```

### Cómo buscarlos

```shell
find / -perm -4000 -type f 2>/dev/null
```

El `4000` es el valor octal del bit SUID. Los binarios normales con SUID son: `su`, `mount`, `umount`, `ping`, `passwd`, `sudo`. Cualquier otro es sospechoso.

---

## Linux Capabilities — la forma moderna y granular

### El problema que resuelven

Root es todo o nada. Si un proceso necesita hacer algo privilegiado, históricamente necesitaba ser root completo con todos sus poderes. Las capabilities dividen los poderes de root en unidades pequeñas y asignan solo lo necesario.

### Analogía

Imagina que root es un rey con poderes absolutos. Las capabilities son delegaciones específicas:
- "Tú puedes abrir puertos < 1024 pero nada más" → `cap_net_bind_service`
- "Tú puedes leer cualquier archivo pero no escribir" → `cap_dac_read_search`
- "Tú puedes cambiar tu UID" → `cap_setuid`

### Las capabilities más importantes en pentesting

| Capability | Qué permite | Cómo explotar |
|------------|-------------|---------------|
| `cap_setuid` | Cambiar el UID del proceso a cualquier valor | Cambiar a UID 0 → root |
| `cap_dac_read_search` | Ignorar permisos de **lectura** | Leer /etc/shadow, claves SSH privadas |
| `cap_dac_override` | Ignorar **todos** los permisos DAC | Leer y escribir cualquier archivo |
| `cap_sys_admin` | Casi todo — muy peligroso | Múltiples vectores de escalada |
| `cap_net_bind_service` | Abrir puertos < 1024 | No útil para escalar |
| `cap_net_raw` | Crear paquetes raw (ping) | No útil para escalar |

### Cómo se ven las capabilities

```shell
getcap -r / 2>/dev/null

/home/cyber/tar cap_dac_read_search=ep
/usr/bin/ping cap_net_raw=ep
```

El sufijo `=ep` significa:
- `e` — **Effective**: la capability está activa cuando el binario se ejecuta
- `p` — **Permitted**: la capability está disponible para usar
- `i` — **Inherited**: se hereda a procesos hijo

### Por qué no aparece con find -perm -4000

Las capabilities son **metadatos del sistema de archivos** almacenados en atributos extendidos (`xattrs`), completamente separados del sistema de permisos tradicional. El `find -perm` solo busca en los bits de permisos estándar y no puede ver los xattrs.

Por eso siempre hay que hacer **ambas búsquedas**:

```shell
find / -perm -4000 -type f 2>/dev/null    # SUID
getcap -r / 2>/dev/null                    # Capabilities
```

### Ejemplo real — cap_dac_read_search en tar

```shell
# tar con cap_dac_read_search puede leer /etc/shadow aunque no tenga permisos
./tar -cvf shadow.tar /etc/shadow
./tar -xvf shadow.tar
cat etc/shadow    # ¡Tienes el hash de root!
```

### Ejemplo real — cap_setuid en python

```shell
# python con cap_setuid puede cambiar su propio UID a 0
python3 -c "import os; os.setuid(0); os.system('/bin/bash')"
# obtienes bash como root
```

---

## El flujo completo de escalada de privilegios

Ahora que entiendes todo esto, el flujo tiene mucho más sentido:

```
Entras como usuario de bajo privilegio (www-data, cyber...)
                    ↓
sudo -l  →  ¿Puedo ejecutar algo como root sin contraseña? → GTFOBins
                    ↓
find / -perm -4000  →  ¿SUID en binarios peligrosos? → GTFOBins
                    ↓
getcap -r /  →  ¿Capabilities peligrosas? (setuid, dac_read, dac_override)
                    ↓
/etc/crontab  →  ¿Scripts ejecutándose como root que pueda modificar?
                    ↓
uname -a  →  ¿Kernel antiguo con exploits públicos?
                    ↓
Credenciales en archivos  →  grep -r "password" /var/www/ /etc/ ~/.bash_history
```

---

## Ejemplo real — Máquina Empire: Breakout

```
cyber (usuario normal, UID 1000)
            ↓
getcap → /home/cyber/tar tiene cap_dac_read_search=ep
            ↓
tar lee /etc/shadow ignorando los permisos (solo root puede leerlo normalmente)
            ↓
John crackea el hash yescrypt ($y$) de root con rockyou.txt
            ↓
su root con la contraseña crackeada → ROOT ✅
```

---

## Referencia rápida — comandos de reconocimiento

```shell
# Permisos y propietario de un archivo
ls -la archivo

# Buscar todos los SUID del sistema
find / -perm -4000 -type f 2>/dev/null

# Buscar todos los GUID del sistema (similar a SUID pero para grupos)
find / -perm -2000 -type f 2>/dev/null

# Buscar capabilities en todos los binarios
getcap -r / 2>/dev/null

# Ver permisos en formato octal
stat archivo

# Ver atributos extendidos (donde viven las capabilities)
getfattr -n security.capability /ruta/binario
```
