# WhatWeb

## Qué es

WhatWeb es una herramienta de reconocimiento web que identifica tecnologías usadas en un sitio web. Con un solo comando es capaz de detectar el CMS, el framework, el servidor web, el lenguaje de programación, las librerías JavaScript, los sistemas de analítica, los plugins instalados y mucho más.

Funciona analizando la respuesta HTTP de la web: cabeceras, código HTML, cookies, scripts y metadatos. Tiene más de 1.800 plugins de detección integrados.

**Cuándo usarla:**
- Como primer paso de reconocimiento web antes de buscar vulnerabilidades
- Para identificar el CMS y buscar exploits específicos (WordPress, Joomla, Drupal...)
- Para detectar versiones de software desactualizadas
- Para mapear la superficie de ataque de un objetivo web

---

## Instalación

```bash
# Kali Linux (ya viene instalado)
whatweb --version

# Debian / Ubuntu
sudo apt install whatweb

# Desde fuente
git clone https://github.com/urbanadventurer/WhatWeb
cd WhatWeb
gem install bundler
bundle install
```

---

## Uso básico

```bash
# Escaneo básico de una URL
whatweb http://192.168.1.10

# Con HTTPS
whatweb https://ejemplo.com

# Múltiples URLs
whatweb http://192.168.1.10 http://192.168.1.20

# Lista de URLs desde fichero
whatweb -i urls.txt
```

**Ejemplo de salida:**
```
http://192.168.1.10 [200 OK] Apache[2.4.48], Country[RESERVED][ZZ],
HTTPServer[Debian Linux][Apache/2.4.48 (Debian)], IP[192.168.1.10],
PHP[7.4.3], WordPress[5.8], X-Powered-By[PHP/7.4.3]
```

---

## Niveles de agresividad

WhatWeb tiene 4 niveles de agresividad que controlan cuántas peticiones hace y cuánta información intenta extraer:

```bash
# Nivel 1 (por defecto) — Una sola petición HTTP. Rápido y sigiloso
whatweb http://192.168.1.10 -a 1

# Nivel 2 — Peticiones adicionales para confirmar detecciones
whatweb http://192.168.1.10 -a 2

# Nivel 3 — Más peticiones, intenta acceder a rutas conocidas de plugins
whatweb http://192.168.1.10 -a 3

# Nivel 4 — Máxima agresividad. Muchas peticiones, puede ser detectado
whatweb http://192.168.1.10 -a 4
```

| Nivel | Peticiones | Uso recomendado |
|-------|-----------|-----------------|
| 1 | 1 | Reconocimiento inicial silencioso |
| 2 | Pocas | Confirmar tecnologías detectadas |
| 3 | Moderadas | Cuando no hay IDS o en entornos controlados |
| 4 | Muchas | CTFs y laboratorios donde no importa el ruido |

---

## Formatos de salida

```bash
# Salida por pantalla (por defecto)
whatweb http://192.168.1.10

# Salida detallada — muestra toda la información de cada plugin
whatweb http://192.168.1.10 -v

# Formato JSON
whatweb http://192.168.1.10 --log-json=resultado.json

# Formato XML
whatweb http://192.168.1.10 --log-xml=resultado.xml

# Formato plano (legible, un resultado por línea)
whatweb http://192.168.1.10 --log-brief=resultado.txt

# Formato verboso a fichero
whatweb http://192.168.1.10 --log-verbose=resultado_verbose.txt

# Formato para importar en Metasploit
whatweb http://192.168.1.10 --log-magictree=resultado_msf.xml

# Múltiples formatos a la vez
whatweb http://192.168.1.10 --log-json=out.json --log-xml=out.xml
```

---

## Escaneo de rangos de red

```bash
# Escanear toda una subred
whatweb http://192.168.1.0/24

# Escanear rango con múltiples hilos
whatweb http://192.168.1.0/24 -t 20

# Escanear solo IPs con el puerto 80 abierto (combinado con masscan)
masscan 192.168.1.0/24 -p 80 --rate 1000 -oL hosts.txt
cat hosts.txt | grep "open" | awk '{print "http://"$4}' > urls.txt
whatweb -i urls.txt
```

---

## Control de hilos y velocidad

```bash
# Número de hilos paralelos (por defecto 25)
whatweb http://192.168.1.0/24 -t 50

# Espera entre peticiones en milisegundos
whatweb http://192.168.1.0/24 --wait=500

# Timeout de conexión en segundos
whatweb http://192.168.1.10 --connect-timeout=10

# Timeout de lectura
whatweb http://192.168.1.10 --read-timeout=30
```

---

## Autenticación y sesiones

```bash
# Autenticación HTTP básica
whatweb http://192.168.1.10 --user usuario:contraseña

# Usar cookie de sesión
whatweb http://192.168.1.10 --cookie "PHPSESSID=abc123; otra=valor"

# Usar fichero de cookies (formato Netscape)
whatweb http://192.168.1.10 --cookiejar cookies.txt

# Guardar cookies recibidas
whatweb http://192.168.1.10 --cookie-jar mis_cookies.txt
```

---

## Cabeceras y proxies

```bash
# Añadir cabecera HTTP personalizada
whatweb http://192.168.1.10 --header "Authorization: Bearer token123"

# Cambiar el User-Agent
whatweb http://192.168.1.10 --user-agent "Mozilla/5.0 (compatible)"

# Usar proxy HTTP
whatweb http://192.168.1.10 --proxy 127.0.0.1:8080

# Usar proxy con autenticación
whatweb http://192.168.1.10 --proxy 127.0.0.1:8080 --proxy-user usuario:pass

# Pasar por Burp Suite
whatweb http://192.168.1.10 --proxy 127.0.0.1:8080
```

---

## Filtrado de resultados

```bash
# Mostrar solo URLs con un código de estado concreto
whatweb http://192.168.1.0/24 --filter-status 200

# Mostrar solo resultados que contengan un plugin específico
whatweb http://192.168.1.0/24 --filter-plugins WordPress

# Ocultar resultados con código 404
whatweb http://192.168.1.0/24 --no-errors

# Mostrar solo errores
whatweb http://192.168.1.0/24 --log-error=errores.txt
```

---

## Plugins

WhatWeb tiene más de 1.800 plugins de detección. Cada plugin detecta una tecnología específica.

```bash
# Listar todos los plugins disponibles
whatweb --list-plugins

# Listar plugins con descripción
whatweb --list-plugins | grep -i wordpress

# Ver información detallada de un plugin
whatweb --info-plugins WordPress

# Usar solo plugins específicos
whatweb http://192.168.1.10 --plugins WordPress,Apache,PHP

# Excluir plugins específicos
whatweb http://192.168.1.10 --disable-plugin GoogleAnalytics

# Cargar plugin personalizado
whatweb http://192.168.1.10 --plugins /ruta/mi_plugin.rb
```

---

## Casos prácticos en pentesting

### Reconocimiento inicial de un objetivo

```bash
# Identificar tecnologías rápidamente
whatweb http://TARGET -v

# Si hay WordPress, buscar versión exacta
whatweb http://TARGET -a 3 -v | grep -i wordpress

# Si hay Apache, buscar versión
whatweb http://TARGET | grep -i apache
```

### Escaneo de múltiples subdominios

```bash
# Crear lista de subdominios (con herramienta como subfinder o amass)
subfinder -d ejemplo.com -o subdominios.txt

# Añadir http:// a cada subdominio
sed 's/^/http:\/\//' subdominios.txt > urls.txt

# Escanear todos con WhatWeb
whatweb -i urls.txt -t 20 --log-json=resultado.json

# Buscar en los resultados los que usan WordPress
cat resultado.json | grep -i wordpress
```

### Detectar versiones desactualizadas

```bash
# Escaneo agresivo para extraer versiones
whatweb http://TARGET -a 3 -v 2>&1 | grep -iE "version|v[0-9]"

# Buscar CMS vulnerables en una red interna
whatweb http://10.0.0.0/24 -t 30 --log-brief=cms.txt
grep -iE "wordpress|joomla|drupal|typo3" cms.txt
```

### Combinación con otras herramientas

```bash
# WhatWeb + Nikto para análisis completo
whatweb http://TARGET -v
nikto -h http://TARGET

# WhatWeb + WPScan si detecta WordPress
whatweb http://TARGET | grep -i wordpress
wpscan --url http://TARGET --enumerate u,p,t

# WhatWeb + Searchsploit para buscar exploits
VERSION=$(whatweb http://TARGET -v | grep "Apache\[" | grep -oP '[\d.]+')
searchsploit apache $VERSION
```

---

## Comparativa con herramientas similares

| Característica | WhatWeb | Wappalyzer | Nmap scripts |
|----------------|---------|------------|--------------|
| Nº de plugins | 1.800+ | 1.500+ | Limitado |
| Velocidad | Alta | Alta | Media |
| Línea de comandos | ✅ | ❌ (extensión) | ✅ |
| Detección de versiones | ✅ | ✅ | Básica |
| Escaneo de rangos | ✅ | ❌ | ✅ |
| Formatos de salida | JSON/XML/txt | JSON | XML |
| Niveles de agresividad | 4 niveles | No | No |

---

## Opciones más usadas — resumen rápido

```bash
-a NIVEL            # Nivel de agresividad (1-4)
-v                  # Salida detallada
-t N                # Número de hilos
-i FICHERO          # Leer URLs desde fichero
--log-json=FILE     # Guardar resultado en JSON
--log-xml=FILE      # Guardar resultado en XML
--log-brief=FILE    # Guardar resultado en texto plano
--proxy HOST:PORT   # Usar proxy
--user USER:PASS    # Autenticación HTTP básica
--cookie "..."      # Enviar cookie de sesión
--user-agent "..."  # Cambiar User-Agent
--no-errors         # Ocultar errores
--list-plugins      # Listar todos los plugins
--wait=MS           # Espera entre peticiones
--connect-timeout=S # Timeout de conexión
```

---

## Notas importantes

- WhatWeb **no escanea vulnerabilidades** — solo identifica tecnologías. Para vulnerabilidades usar Nikto o herramientas específicas del CMS detectado.
- A nivel de agresividad 1 puede dar **falsos negativos** — tecnologías que no detecta porque necesita más peticiones para confirmarlas.
- En webs con **WAF** activo, niveles altos de agresividad pueden hacer que la IP quede bloqueada.
- La detección de versiones **no siempre es exacta** — depende de si la web expone esa información en las cabeceras o el HTML.
- Combinar siempre con **WPScan** si detecta WordPress, **Droopescan** si detecta Drupal o **JoomScan** si detecta Joomla para un análisis más profundo.
