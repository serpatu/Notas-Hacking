# Enumeración Web (Puerto 80/443)

## Descripción

Checklist completo para cuando encuentras un servidor web abierto. Sigue el orden de arriba a abajo.

---

## Paso 1 — Visita la web manualmente

Abre el navegador y visita `http://TARGET_IP`. Busca:

- Tecnología usada (WordPress, Joomla, aplicación custom)
- Versión visible en el pie de página
- Formularios de login
- Comentarios en el código fuente (`Ctrl+U`)
- Rutas mencionadas en el HTML

---

## Paso 2 — Enumera rutas con Gobuster

```shell
# Enumeración básica de directorios
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/dirb/common.txt

# Con extensiones de archivo
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak

# Wordlist más completa (más lenta pero más exhaustiva)
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### Qué buscar en los resultados de Gobuster

| Ruta | Qué significa |
|------|---------------|
| `/admin`, `/administrator` | Panel de administración |
| `/phpinfo` | Información crítica del servidor |
| `/phpMyAdmin` | Gestión de base de datos |
| `/backup`, `/bak` | Posibles backups con info sensible |
| `/upload`, `/uploads` | Directorio de subida de archivos |
| `.git` | Repositorio Git expuesto |
| `/config`, `/conf` | Archivos de configuración |
| Cualquier CMS conocido | TikiWiki, TWiki, WordPress... |

---

## Paso 3 — Identifica el CMS o aplicación

Si encuentras una aplicación conocida:

```shell
searchsploit nombre_aplicacion
searchsploit nombre_aplicacion version
```

Busca también en:
- `https://book.hacktricks.xyz/network-services-pentesting/pentesting-web`
- `https://www.exploit-db.com`

---

## Paso 4 — Analiza phpinfo si existe

Si encuentras `/phpinfo`, anota siempre:

- **PHP Version** — para buscar exploits de PHP
- **DOCUMENT_ROOT** — ruta absoluta del servidor (necesaria para LFI)
- **disable_functions** — qué funciones PHP están bloqueadas
- **SERVER_ADDR / SERVER_NAME** — información del servidor

---

## Paso 5 — Prueba credenciales por defecto

Si encuentras un panel de login, prueba siempre antes de hacer fuerza bruta:

| Aplicación | Usuario | Contraseña |
|------------|---------|------------|
| WordPress | admin | admin / password |
| Joomla | admin | admin |
| TikiWiki | admin | admin |
| TWiki | admin | admin |
| Tomcat | admin/tomcat | admin/tomcat/s3cret |
| phpMyAdmin | root | (vacía) / root / toor |
| DVWA | admin | password |

---

## Paso 6 — Busca vulnerabilidades comunes

### LFI (Local File Inclusion)

Busca parámetros que carguen archivos:

```
http://TARGET/page.php?file=../../../../etc/passwd
http://TARGET/page.php?include=../../../etc/passwd
http://TARGET/page.php?page=../../../etc/passwd
```

Si devuelve el contenido de `/etc/passwd`, hay LFI.

### SQLi básica

Añade una comilla simple a parámetros GET/POST:

```
http://TARGET/page.php?id=1'
```

Si devuelve error SQL, prueba con SQLMap:

```shell
sqlmap -u "http://TARGET/page.php?id=1" --dbs --batch
```

### RCE en parámetros

Prueba metacaracteres de shell:

```
http://TARGET/page.php?cmd=;id;
http://TARGET/page.php?cmd=|id
http://TARGET/page.php?cmd=`id`
```

---

## Paso 7 — Fuerza bruta si hay login

Si las credenciales por defecto no funcionan y no hay otro vector:

```shell
# Con Hydra para formularios web
hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET_IP http-post-form "/login.php:user=^USER^&pass=^PASS^:Invalid"

# Con Hydra para HTTP Basic Auth
hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET_IP http-get /admin
```

---

## Paso 8 — Directorios de usuario con ~ (Apache mod_userdir)

Si Apache tiene el módulo `mod_userdir` activo, la ruta `/~nombre` sirve el contenido de `/home/nombre/public_html/`. Esto puede revelar directorios con información sensible.

> **Regla importante:** Cuando fuzzes con `~FUZZ`, usa siempre primero una **wordlist de directorios genéricos**, no de usuarios. El nombre después del `~` puede ser cualquier palabra, no necesariamente un usuario del sistema. Solo usa wordlists de usuarios si tienes evidencia clara de que el servidor tiene `mod_userdir` activo con cuentas reales.

```shell
# Buscar directorios ~nombre con ffuf (más rápido que gobuster para esto)
ffuf -u "http://TARGET_IP/~FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200,301,302 \
  -t 100

# Incluir 403 — Apache devuelve 403 cuando el directorio existe pero no tienes acceso
# Primero averigua el tamaño de una respuesta 404 normal y filtra por ese tamaño
ffuf -u "http://TARGET_IP/~FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200,301,302,403 \
  -fs TAMAÑO_RESPUESTA_404 \
  -t 100
```

### Cuándo usar cada wordlist

| Situación | Wordlist recomendada |
|-----------|---------------------|
| Directorios generales y `~FUZZ` | `dirbuster/directory-list-2.3-medium.txt` ⭐ |
| Archivos web comunes | `seclists/Discovery/Web-Content/common.txt` |
| Nombres de usuario reales | `seclists/Usernames/Names/names.txt` |
| Archivos ocultos dentro de un directorio | `directory-list-2.3-medium.txt` con prefijo `.` |

### Buscar archivos ocultos dentro de un directorio

En Linux los archivos ocultos empiezan por `.`. Para encontrarlos usa ffuf con el punto como prefijo:

```shell
ffuf -u "http://TARGET_IP/~secret/.FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -e .txt,.php,.html \
  -mc 200 \
  -t 100
```

---

## Árbol de decisión rápido

```
Puerto 80 abierto
      ↓
Gobuster → ¿Qué rutas hay?
      ↓
/phpinfo → Anotar versión PHP y DOCUMENT_ROOT
      ↓
CMS conocido → searchsploit → ¿Exploit público?
      ↓                              SÍ → Explotar
      ↓                              NO → Credenciales por defecto
      ↓
Panel login → Credenciales por defecto → Fuerza bruta
      ↓
Parámetros en URLs → Probar LFI / SQLi / RCE
```
