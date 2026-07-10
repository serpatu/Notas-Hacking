# Wfuzz

## Qué es

Wfuzz es una herramienta de fuzzing web escrita en Python. Su función principal es sustituir cualquier parte de una petición HTTP por valores de una wordlist y analizar las respuestas para encontrar recursos ocultos, parámetros vulnerables, credenciales válidas y mucho más.

A diferencia de herramientas como ffuf o Gobuster que están optimizadas para descubrimiento de directorios, Wfuzz es mucho más flexible: puede fuzzear cualquier parte de la petición HTTP — la URL, las cabeceras, el cuerpo POST, las cookies, los parámetros GET — e incluso varios puntos a la vez con múltiples wordlists simultáneas.

**Cuándo usarla:**
- Descubrimiento de directorios y archivos ocultos
- Fuzzing de parámetros GET y POST
- Ataques de fuerza bruta a formularios de login
- Descubrimiento de subdominios
- Fuzzing de cabeceras HTTP
- Fuzzing de cookies
- Cualquier caso donde ffuf o Gobuster se quedan cortos por falta de flexibilidad

---

## Instalación

```bash
# Kali Linux (ya viene instalado)
wfuzz --version

# pip
pip install wfuzz --break-system-packages

# Desde fuente
git clone https://github.com/xmendez/wfuzz
cd wfuzz
python3 setup.py install
```

---

## Concepto fundamental — FUZZ

El marcador `FUZZ` es la palabra clave de Wfuzz. Se coloca en cualquier parte de la petición HTTP y Wfuzz lo sustituye por cada línea de la wordlist. Si necesitas fuzzear varios puntos a la vez se usan `FUZ2Z`, `FUZ3Z`, etc.

```bash
# FUZZ en la URL (descubrimiento de directorios)
wfuzz -w wordlist.txt http://TARGET/FUZZ

# FUZZ en un parámetro GET
wfuzz -w wordlist.txt "http://TARGET/page.php?id=FUZZ"

# FUZZ en el cuerpo POST
wfuzz -w wordlist.txt -d "user=admin&pass=FUZZ" http://TARGET/login.php

# FUZZ en una cabecera
wfuzz -w wordlist.txt -H "X-Custom-Header: FUZZ" http://TARGET/

# Dos FUZZ simultáneos (dos wordlists)
wfuzz -w usuarios.txt -w passwords.txt -d "user=FUZZ&pass=FUZ2Z" http://TARGET/login
```

---

## Sintaxis básica

```bash
wfuzz [OPCIONES] -w WORDLIST URL_CON_FUZZ
```

```bash
# Descubrimiento de directorios básico
wfuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://192.168.1.10/FUZZ

# Con extensiones de archivo
wfuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://192.168.1.10/FUZZ.php

# Mostrando solo respuestas con código 200
wfuzz -w wordlist.txt --hc 404 http://192.168.1.10/FUZZ
```

---

## Wordlists

```bash
# Especificar wordlist
wfuzz -w /ruta/wordlist.txt http://TARGET/FUZZ

# Múltiples wordlists (una por cada marcador FUZZ)
wfuzz -w wordlist1.txt -w wordlist2.txt http://TARGET/FUZZ/FUZ2Z

# Wordlist inline (lista de valores directamente en el comando)
wfuzz -z list,admin-user-root-test http://TARGET/FUZZ

# Wordlist de rango numérico
wfuzz -z range,1-100 "http://TARGET/page.php?id=FUZZ"

# Wordlist de rango con relleno de ceros
wfuzz -z range,00-99 "http://TARGET/item=FUZZ"

# Combinación de wordlists (producto cartesiano)
wfuzz -w users.txt -w passes.txt --hc 302 -d "u=FUZZ&p=FUZ2Z" http://TARGET/login
```

**Wordlists recomendadas en Kali:**

```bash
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt   # Directorios
/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt    # Directorios (pequeña)
/usr/share/seclists/Discovery/Web-Content/common.txt           # Archivos comunes
/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt # Archivos
/usr/share/seclists/Usernames/top-usernames-shortlist.txt      # Usuarios
/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt  # Passwords
/usr/share/wordlists/fasttrack.txt                             # Passwords CTF
/usr/share/wordlists/rockyou.txt                               # Passwords grande
```

---

## Filtrado de respuestas

El filtrado es la parte más importante de Wfuzz. Sin filtros, muestra todas las respuestas y es imposible encontrar lo que buscas. Con filtros, solo muestra las respuestas que te interesan.

### Filtrar por código de respuesta HTTP

```bash
# Ocultar respuestas con código 404 (las más comunes en dir fuzzing)
wfuzz -w wordlist.txt --hc 404 http://TARGET/FUZZ

# Ocultar múltiples códigos
wfuzz -w wordlist.txt --hc 404,403,500 http://TARGET/FUZZ

# Mostrar SOLO respuestas con código 200
wfuzz -w wordlist.txt --sc 200 http://TARGET/FUZZ

# Mostrar solo 200 y 301
wfuzz -w wordlist.txt --sc 200,301 http://TARGET/FUZZ
```

### Filtrar por número de líneas

```bash
# Ocultar respuestas con 9 líneas (páginas de error genéricas)
wfuzz -w wordlist.txt --hl 9 http://TARGET/FUZZ

# Ocultar múltiples cantidades de líneas
wfuzz -w wordlist.txt --hl 9,15 http://TARGET/FUZZ

# Mostrar solo respuestas con 50 líneas
wfuzz -w wordlist.txt --sl 50 http://TARGET/FUZZ
```

### Filtrar por número de palabras

```bash
# Ocultar respuestas con 12 palabras
wfuzz -w wordlist.txt --hw 12 http://TARGET/FUZZ

# Mostrar solo respuestas con más de 100 palabras
wfuzz -w wordlist.txt --sw 100 http://TARGET/FUZZ
```

### Filtrar por tamaño de respuesta (bytes)

```bash
# Ocultar respuestas de 1234 bytes (tamaño de página de error)
wfuzz -w wordlist.txt --hh 1234 http://TARGET/FUZZ

# Mostrar solo respuestas de un tamaño concreto
wfuzz -w wordlist.txt --sh 5000 http://TARGET/FUZZ
```

### Filtros con expresiones regulares

```bash
# Ocultar respuestas que contengan "Not Found"
wfuzz -w wordlist.txt --hh "Not Found" http://TARGET/FUZZ

# Mostrar solo respuestas que contengan "Welcome"
wfuzz -w wordlist.txt --ss "Welcome" http://TARGET/FUZZ

# Mostrar solo respuestas que contengan "admin" o "panel"
wfuzz -w wordlist.txt --ss "admin|panel" http://TARGET/FUZZ
```

### Tabla resumen de filtros

| Flag | Acción | Filtra por |
|------|--------|-----------|
| `--hc` | Ocultar | Código HTTP |
| `--sc` | Mostrar | Código HTTP |
| `--hl` | Ocultar | Nº de líneas |
| `--sl` | Mostrar | Nº de líneas |
| `--hw` | Ocultar | Nº de palabras |
| `--sw` | Mostrar | Nº de palabras |
| `--hh` | Ocultar | Nº de bytes / string |
| `--sh` | Mostrar | Nº de bytes / string |

---

## Métodos HTTP

```bash
# GET (por defecto)
wfuzz -w wordlist.txt http://TARGET/FUZZ

# POST
wfuzz -w wordlist.txt -d "param=FUZZ" http://TARGET/endpoint

# PUT
wfuzz -w wordlist.txt -X PUT -d "data=FUZZ" http://TARGET/api/FUZZ

# DELETE
wfuzz -w wordlist.txt -X DELETE http://TARGET/api/FUZZ

# PATCH
wfuzz -w wordlist.txt -X PATCH -d "field=FUZZ" http://TARGET/api/item/1
```

---

## Fuzzing de parámetros GET

```bash
# Fuzzear el valor de un parámetro
wfuzz -w wordlist.txt --hc 404 "http://TARGET/page.php?id=FUZZ"

# Fuzzear el nombre del parámetro
wfuzz -w wordlist.txt --hc 404 "http://TARGET/page.php?FUZZ=1"

# Fuzzear múltiples parámetros
wfuzz -w ids.txt -w nombres.txt --hc 404 "http://TARGET/page.php?id=FUZZ&name=FUZ2Z"

# Buscar parámetros ocultos (con lista de parámetros comunes)
wfuzz -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
      --hh 1234 "http://TARGET/page.php?FUZZ=test"
```

---

## Fuzzing de formularios POST

```bash
# Login básico fuzzeando la contraseña
wfuzz -w /usr/share/wordlists/fasttrack.txt \
      -d "username=admin&password=FUZZ" \
      --hc 302 http://TARGET/login.php

# Login fuzzeando usuario y contraseña a la vez
wfuzz -w usuarios.txt -w passwords.txt \
      -d "user=FUZZ&pass=FUZ2Z" \
      --hc 200 http://TARGET/login

# Con token CSRF (valor fijo obtenido previamente)
wfuzz -w wordlist.txt \
      -d "username=admin&password=FUZZ&csrf_token=abc123" \
      --sc 302 http://TARGET/login

# Login con JSON
wfuzz -w passwords.txt \
      -H "Content-Type: application/json" \
      -d '{"user":"admin","pass":"FUZZ"}' \
      --sc 200 http://TARGET/api/login
```

---

## Fuzzing de cabeceras HTTP

```bash
# Fuzzear el valor de una cabecera
wfuzz -w wordlist.txt -H "X-Forwarded-For: FUZZ" http://TARGET/admin

# Fuzzear el User-Agent
wfuzz -w user-agents.txt -H "User-Agent: FUZZ" http://TARGET/

# Fuzzear el Host (descubrimiento de virtual hosts)
wfuzz -w subdominios.txt -H "Host: FUZZ.ejemplo.com" --hh 1234 http://IP/

# Fuzzear el valor de Authorization
wfuzz -w tokens.txt -H "Authorization: Bearer FUZZ" --sc 200 http://TARGET/api/data

# Fuzzear múltiples cabeceras
wfuzz -w wordlist.txt \
      -H "X-Custom: FUZZ" \
      -H "User-Agent: Mozilla/5.0" \
      http://TARGET/
```

---

## Fuzzing de cookies

```bash
# Fuzzear el valor de una cookie
wfuzz -w wordlist.txt -b "session=FUZZ" http://TARGET/dashboard

# Fuzzear múltiples cookies
wfuzz -w wordlist.txt -b "user=FUZZ; role=admin" http://TARGET/admin

# Fuzzear el nombre de la cookie
wfuzz -w cookie-names.txt -b "FUZZ=valor" http://TARGET/
```

---

## Descubrimiento de subdominios

```bash
# Subdominios fuzzeando la cabecera Host
wfuzz -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
      -H "Host: FUZZ.ejemplo.com" \
      --hh 1234 \
      http://IP_DEL_SERVIDOR/

# Con código de estado
wfuzz -w subdominios.txt \
      -H "Host: FUZZ.ejemplo.com" \
      --sc 200,301,302 \
      http://IP/
```

---

## Descubrimiento de archivos con extensiones

```bash
# Buscar archivos PHP
wfuzz -w wordlist.txt --hc 404 http://TARGET/FUZZ.php

# Buscar archivos de backup y configuración
wfuzz -w wordlist.txt --hc 404 http://TARGET/FUZZ.bak
wfuzz -w wordlist.txt --hc 404 http://TARGET/FUZZ.old
wfuzz -w wordlist.txt --hc 404 http://TARGET/FUZZ.conf
wfuzz -w wordlist.txt --hc 404 http://TARGET/FUZZ.txt

# Múltiples extensiones con una sola wordlist
wfuzz -w wordlist.txt -z list,.php-.html-.txt-.bak \
      --hc 404 http://TARGET/FUZZFUZ2Z

# Con la extensión integrada en la wordlist
wfuzz -w /usr/share/seclists/Discovery/Web-Content/raft-large-files.txt \
      --hc 404 http://TARGET/FUZZ
```

---

## Proxies y SSL

```bash
# Pasar tráfico por Burp Suite
wfuzz -w wordlist.txt --hc 404 \
      -p 127.0.0.1:8080 \
      http://TARGET/FUZZ

# Proxy con autenticación
wfuzz -w wordlist.txt \
      -p 127.0.0.1:8080:usuario:password \
      http://TARGET/FUZZ

# Ignorar errores de certificado SSL
wfuzz -w wordlist.txt --hc 404 \
      https://TARGET/FUZZ

# Certificado SSL cliente
wfuzz -w wordlist.txt \
      --cert certificado.pem \
      https://TARGET/FUZZ
```

---

## Autenticación

```bash
# Autenticación HTTP básica
wfuzz -w wordlist.txt \
      --basic admin:password \
      http://TARGET/FUZZ

# Autenticación NTLM (entornos Windows/Active Directory)
wfuzz -w wordlist.txt \
      --ntlm dominio\\usuario:password \
      http://TARGET/FUZZ

# Autenticación Digest
wfuzz -w wordlist.txt \
      --digest usuario:password \
      http://TARGET/FUZZ

# Fuerza bruta de autenticación básica
wfuzz -w usuarios.txt -w passwords.txt \
      --basic FUZZ:FUZ2Z \
      --hc 401 http://TARGET/admin/
```

---

## Control de velocidad y conexiones

```bash
# Número de hilos concurrentes (por defecto 10)
wfuzz -w wordlist.txt --hc 404 -t 50 http://TARGET/FUZZ

# Espera entre peticiones en segundos
wfuzz -w wordlist.txt --hc 404 -s 0.5 http://TARGET/FUZZ

# Timeout de conexión
wfuzz -w wordlist.txt --hc 404 --conn-delay 10 http://TARGET/FUZZ

# Timeout de recepción
wfuzz -w wordlist.txt --hc 404 --req-delay 10 http://TARGET/FUZZ

# Máximo de retries por petición fallida
wfuzz -w wordlist.txt --hc 404 -R 3 http://TARGET/FUZZ
```

---

## Formato de salida y guardado

```bash
# Guardar resultado en fichero de texto
wfuzz -w wordlist.txt --hc 404 -o resultado.txt http://TARGET/FUZZ

# Guardar en formato JSON
wfuzz -w wordlist.txt --hc 404 -f resultado.json,json http://TARGET/FUZZ

# Guardar en formato HTML
wfuzz -w wordlist.txt --hc 404 -f resultado.html,html http://TARGET/FUZZ

# Modo silencioso (solo muestra resultados, sin banner)
wfuzz -w wordlist.txt --hc 404 -q http://TARGET/FUZZ

# Modo verbose (muestra todas las peticiones)
wfuzz -w wordlist.txt -v http://TARGET/FUZZ
```

---

## Payloads especiales

Wfuzz tiene generadores de payloads integrados además de las wordlists:

```bash
# Lista de valores
wfuzz -z list,uno-dos-tres http://TARGET/FUZZ

# Rango numérico
wfuzz -z range,1-1000 "http://TARGET/?id=FUZZ"

# Rango con paso personalizado
wfuzz -z range,0-100-2 "http://TARGET/?id=FUZZ"  # de 2 en 2

# Hex range
wfuzz -z hexrange,00-ff "http://TARGET/?char=FUZZ"

# Permutaciones de caracteres
wfuzz -z permutation,abc-2 http://TARGET/FUZZ  # aa ab ac ba bb bc ca cb cc

# Encoding de payloads
wfuzz -w wordlist.txt -e urlencode "http://TARGET/?q=FUZZ"

# Ver todos los encoders disponibles
wfuzz -e encoders

# Ver todos los tipos de payload disponibles
wfuzz -e payloads
```

---

## Encoders

Los encoders transforman el valor de FUZZ antes de enviarlo. Se aplican con la sintaxis `FUZZ~encoder`:

```bash
# URL encoding
wfuzz -w wordlist.txt "http://TARGET/FUZZ~urlencode"

# Base64
wfuzz -w wordlist.txt "http://TARGET/FUZZ~base64"

# HTML encoding
wfuzz -w wordlist.txt "http://TARGET/FUZZ~html"

# MD5
wfuzz -w wordlist.txt "http://TARGET/FUZZ~md5"

# SHA1
wfuzz -w wordlist.txt "http://TARGET/FUZZ~sha1"

# Doble URL encoding
wfuzz -w wordlist.txt "http://TARGET/FUZZ~urlencode~urlencode"

# Ver todos los encoders
wfuzz -e encoders
```

---

## Casos prácticos en pentesting

### Descubrimiento inicial de directorios

```bash
# Escaneo rápido con wordlist media
wfuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
      --hc 404 -t 40 \
      http://TARGET/FUZZ
```

### Buscar paneles de administración

```bash
wfuzz -w /usr/share/seclists/Discovery/Web-Content/common.txt \
      --hc 404 --sc 200,301,302 \
      http://TARGET/FUZZ
```

### SQL Injection básico en parámetro GET

```bash
# Payload de SQLi
cat > sqli.txt << EOF
'
''
' OR '1'='1
' OR 1=1--
1' ORDER BY 1--
1' UNION SELECT NULL--
EOF

wfuzz -w sqli.txt --hh 1234 "http://TARGET/page.php?id=FUZZ"
```

### LFI (Local File Inclusion)

```bash
wfuzz -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
      --hh 1234 \
      "http://TARGET/page.php?file=FUZZ"
```

### XSS básico

```bash
cat > xss.txt << EOF
<script>alert(1)</script>
"><script>alert(1)</script>
'><script>alert(1)</script>
<img src=x onerror=alert(1)>
EOF

wfuzz -w xss.txt --hh 1234 "http://TARGET/search.php?q=FUZZ"
```

### Fuerza bruta a login WordPress

```bash
wfuzz -w /usr/share/wordlists/fasttrack.txt \
      -d "log=admin&pwd=FUZZ&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1" \
      -b "wordpress_test_cookie=WP+Cookie+check" \
      --hh 3828 \
      http://TARGET/wp-login.php
```

### API REST fuzzing

```bash
# Descubrir endpoints de API
wfuzz -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
      --hc 404 \
      http://TARGET/api/v1/FUZZ

# Fuzzear versión de la API
wfuzz -z range,1-10 --hc 404 http://TARGET/api/vFUZZ/users

# Fuzzear IDs de recursos
wfuzz -z range,1-1000 --hc 404 http://TARGET/api/users/FUZZ
```

---

## Comparativa con ffuf y Gobuster

| Característica | Wfuzz | ffuf | Gobuster |
|----------------|-------|------|----------|
| Velocidad | Media | Alta | Alta |
| Flexibilidad | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Fuzzing de cabeceras | ✅ | ✅ | ❌ |
| Fuzzing de cookies | ✅ | ✅ | ❌ |
| Múltiples FUZZ | ✅ | ✅ | ❌ |
| Encoders integrados | ✅ | Limitado | ❌ |
| Payloads especiales | ✅ | ❌ | ❌ |
| Facilidad de uso | Media | Alta | Alta |
| Filtros | Muy completos | Completos | Básicos |

---

## Opciones más usadas — resumen rápido

```bash
-w WORDLIST         # Wordlist a usar
-z PAYLOAD          # Tipo de payload (list, range, etc.)
-d "datos"          # Cuerpo de la petición POST
-H "cabecera"       # Cabecera HTTP personalizada
-b "cookie"         # Cookie
-X MÉTODO           # Método HTTP (GET, POST, PUT...)
-t N                # Número de hilos
-s N                # Segundos de espera entre peticiones
-p HOST:PORT        # Proxy
-o FICHERO          # Guardar resultado
-f FICHERO,FORMATO  # Guardar en formato específico
-q                  # Modo silencioso
-v                  # Modo verbose
-R N                # Reintentos
--hc CÓDIGO         # Ocultar por código HTTP
--sc CÓDIGO         # Mostrar por código HTTP
--hl N              # Ocultar por nº líneas
--sl N              # Mostrar por nº líneas
--hw N              # Ocultar por nº palabras
--sw N              # Mostrar por nº palabras
--hh N/STR          # Ocultar por bytes o string
--sh N/STR          # Mostrar por bytes o string
--basic USER:PASS   # Autenticación básica
--ntlm DOM\\U:PASS  # Autenticación NTLM
-e encoders         # Listar encoders disponibles
-e payloads         # Listar tipos de payload
```

---

## Notas importantes

- Wfuzz **es más lento que ffuf** — para escaneos grandes de directorios ffuf es mejor opción. Wfuzz brilla en escenarios que requieren flexibilidad.
- El **filtrado correcto** es clave — antes de lanzar un escaneo grande, haz una petición manual para ver cómo responde el servidor y qué código/tamaño/líneas tienen las respuestas negativas.
- Con **múltiples wordlists** el número de peticiones se multiplica (producto cartesiano) — con dos wordlists de 1.000 líneas se hacen 1.000.000 de peticiones.
- En entornos con **rate limiting** usar `-s` para añadir espera entre peticiones.
- Siempre **revisar el banner de respuesta** de la primera petición para ajustar los filtros antes del escaneo completo.
