# Masscan

## Qué es

Masscan es el escáner de puertos más rápido del mundo. Puede escanear todo el espacio de direcciones IPv4 de Internet (4.000 millones de IPs) en menos de 6 minutos enviando 10 millones de paquetes por segundo. Fue creado por Robert Graham y está escrito en C.

A diferencia de Nmap, que establece conexiones TCP completas, Masscan implementa su propia pila TCP/IP asíncrona. Esto le permite enviar paquetes SYN a una velocidad masiva sin esperar respuesta, y procesar las respuestas de forma independiente. El resultado es una velocidad de escaneo entre 100 y 1000 veces mayor que Nmap.

**Cuándo usar Masscan en lugar de Nmap:**
- Cuando necesitas escanear rangos de red muy grandes rápidamente
- Para descubrir qué hosts tienen un puerto concreto abierto en toda una red
- Como primer paso de reconocimiento para luego profundizar con Nmap en los hosts encontrados

**Cuándo usar Nmap en lugar de Masscan:**
- Cuando necesitas detección de versiones de servicios (-sV)
- Cuando necesitas ejecutar scripts NSE
- Cuando necesitas resultados precisos en redes con pérdida de paquetes

---

## Instalación

```bash
# Kali Linux (ya viene instalado)
masscan --version

# Debian / Ubuntu
sudo apt install masscan

# Compilar desde fuente
git clone https://github.com/robertdavidgraham/masscan
cd masscan
make
sudo make install
```

---

## Sintaxis básica

```bash
masscan [OPCIONES] IP/RANGO -p PUERTOS
```

**Importante:** Masscan requiere privilegios de root para crear sockets raw.

```bash
# Escaneo básico de un puerto
sudo masscan 192.168.1.0/24 -p 80

# Escaneo de múltiples puertos
sudo masscan 192.168.1.0/24 -p 80,443,22,21

# Escaneo de un rango de puertos
sudo masscan 192.168.1.0/24 -p 1-1000

# Escaneo de todos los puertos
sudo masscan 192.168.1.0/24 -p 0-65535
```

---

## Control de velocidad

La velocidad de transmisión es el parámetro más importante de Masscan. Se controla con `--rate` y se mide en paquetes por segundo (pps).

```bash
# Velocidad conservadora — redes domésticas o VPNs
sudo masscan 192.168.1.0/24 -p 80 --rate 100

# Velocidad media — redes locales
sudo masscan 10.0.0.0/8 -p 80 --rate 10000

# Velocidad alta — conexiones rápidas
sudo masscan 10.0.0.0/8 -p 80 --rate 100000

# Velocidad máxima — conexiones de fibra / datacenters
sudo masscan 0.0.0.0/0 -p 80 --rate 1000000
```

**Recomendaciones de velocidad:**

| Entorno | Rate recomendado |
|---------|-----------------|
| Red doméstica / WiFi | 100 - 1.000 |
| LAN corporativa | 10.000 - 100.000 |
| Datacenter / fibra | 100.000 - 1.000.000 |
| Escaneo de Internet | hasta 10.000.000 |

A velocidades altas Masscan puede saturar la red y generar falsos negativos (puertos abiertos que no detecta). Si la precisión importa, baja la velocidad.

---

## Especificación de IPs y rangos

```bash
# IP individual
sudo masscan 192.168.1.1 -p 80

# Rango CIDR
sudo masscan 192.168.1.0/24 -p 80

# Múltiples rangos
sudo masscan 192.168.1.0/24 192.168.2.0/24 -p 80

# Rango con guión
sudo masscan 192.168.1.1-192.168.1.254 -p 80

# Lista de IPs desde fichero
sudo masscan -iL targets.txt -p 80

# Excluir IPs (muy importante para evitar targets sensibles)
sudo masscan 10.0.0.0/8 -p 80 --exclude 10.0.0.1

# Excluir desde fichero
sudo masscan 10.0.0.0/8 -p 80 --excludefile excluidos.txt

# Todo Internet (cuidado — solo en entornos autorizados)
sudo masscan 0.0.0.0/0 -p 80 --rate 100000
```

---

## Especificación de puertos

```bash
# Puerto único
sudo masscan 192.168.1.0/24 -p 443

# Lista de puertos
sudo masscan 192.168.1.0/24 -p 22,80,443,3389,8080

# Rango de puertos
sudo masscan 192.168.1.0/24 -p 1-1024

# Combinación de rangos y puertos individuales
sudo masscan 192.168.1.0/24 -p 22,80-100,443,8000-9000

# Todos los puertos
sudo masscan 192.168.1.0/24 -p 0-65535

# Puertos UDP (prefijo U:)
sudo masscan 192.168.1.0/24 -pU:53,161

# Mezcla TCP y UDP
sudo masscan 192.168.1.0/24 -p 80,U:53
```

---

## Protocolos y tipos de escaneo

Por defecto Masscan envía paquetes SYN TCP. También soporta otros tipos:

```bash
# SYN scan TCP (por defecto) — detecta puertos abiertos
sudo masscan 192.168.1.0/24 -p 80

# UDP scan — más lento, para servicios UDP
sudo masscan 192.168.1.0/24 -pU:53,161,162,500

# ICMP ping — verificar qué hosts están activos
sudo masscan 192.168.1.0/24 --ping

# SCTP — protocolo alternativo a TCP usado en telecomunicaciones
sudo masscan 192.168.1.0/24 -pS:80
```

---

## Interfaz de red y configuración

```bash
# Especificar interfaz de red
sudo masscan 192.168.1.0/24 -p 80 --interface eth0

# Especificar IP de origen
sudo masscan 192.168.1.0/24 -p 80 --source-ip 192.168.1.100

# Especificar puerto de origen
sudo masscan 192.168.1.0/24 -p 80 --source-port 12345

# Especificar MAC del router (gateway) — necesario en algunos entornos
sudo masscan 192.168.1.0/24 -p 80 --router-mac 00:11:22:33:44:55

# Especificar adaptador de red en Windows
masscan 192.168.1.0/24 -p 80 --adapter "Intel(R) Ethernet"
```

---

## Formatos de salida

```bash
# Salida por pantalla (por defecto)
sudo masscan 192.168.1.0/24 -p 80

# Guardar en formato XML
sudo masscan 192.168.1.0/24 -p 80 -oX resultado.xml

# Guardar en formato grepable (similar a Nmap -oG)
sudo masscan 192.168.1.0/24 -p 80 -oG resultado.gnmap

# Guardar en formato JSON
sudo masscan 192.168.1.0/24 -p 80 -oJ resultado.json

# Guardar en formato binario (para reanudar escaneos)
sudo masscan 192.168.1.0/24 -p 80 -oB resultado.bin

# Guardar en formato lista simple (solo IP:puerto)
sudo masscan 192.168.1.0/24 -p 80 -oL resultado.txt

# Leer resultado binario y convertir
sudo masscan --readscan resultado.bin -oX convertido.xml
```

**Ejemplo de salida en formato lista (-oL):**
```
open tcp 80 192.168.1.10 1609459200
open tcp 80 192.168.1.15 1609459200
open tcp 80 192.168.1.22 1609459200
```

---

## Reanudar escaneos interrumpidos

Masscan guarda automáticamente el estado del escaneo en un fichero llamado `paused.conf`. Si el escaneo se interrumpe, se puede reanudar:

```bash
# Pausar un escaneo — Ctrl+C genera paused.conf automáticamente

# Reanudar desde donde se quedó
sudo masscan --resume paused.conf

# Guardar estado en fichero personalizado
sudo masscan 10.0.0.0/8 -p 80 --resume-filename mi_estado.conf

# Ver el progreso durante el escaneo
sudo masscan 10.0.0.0/8 -p 80 --status-file estado.txt
```

El fichero `paused.conf` contiene el rango restante por escanear, la velocidad configurada y otros parámetros del escaneo. Es muy útil para escaneos de rangos grandes que pueden tardar horas.

---

## Banners y detección de servicios

Masscan puede capturar banners de servicios — la respuesta inicial que envían algunos servicios al conectarse. Esto permite identificar versiones de software sin usar Nmap.

```bash
# Capturar banner HTTP
sudo masscan 192.168.1.0/24 -p 80 --banners

# Capturar banner HTTPS (requiere configuración SSL)
sudo masscan 192.168.1.0/24 -p 443 --banners

# Capturar banner SSH
sudo masscan 192.168.1.0/24 -p 22 --banners

# Capturar banner FTP
sudo masscan 192.168.1.0/24 -p 21 --banners

# Capturar banner SMTP
sudo masscan 192.168.1.0/24 -p 25 --banners

# Especificar tiempo de espera para recibir el banner
sudo masscan 192.168.1.0/24 -p 80 --banners --connection-timeout 10
```

La captura de banners es más lenta que el escaneo SYN puro porque requiere completar la conexión TCP. Para velocidades muy altas, desactiva los banners.

---

## Fichero de configuración

En lugar de pasar todos los parámetros por línea de comandos, se puede usar un fichero de configuración:

```bash
# Generar plantilla de configuración
sudo masscan --echo > masscan.conf

# Usar fichero de configuración
sudo masscan -c masscan.conf

# Combinar fichero de configuración con parámetros adicionales
sudo masscan -c masscan.conf --rate 50000
```

**Ejemplo de masscan.conf:**
```
rate = 10000.00
output-format = xml
output-filename = resultado.xml
ports = 80,443,22,21,3389
range = 192.168.1.0/24
interface = eth0
```

Los parámetros del fichero de configuración tienen los mismos nombres que las opciones de línea de comandos pero sin los guiones.

---

## Combinación con Nmap (flujo de trabajo habitual)

El flujo más eficiente en pentesting es usar Masscan para el descubrimiento rápido y Nmap para el análisis detallado de los hosts encontrados:

```bash
# Paso 1 — Masscan: descubrir todos los hosts con puertos abiertos rápidamente
sudo masscan 10.0.0.0/8 -p 1-65535 --rate 100000 -oL hosts_abiertos.txt

# Paso 2 — Extraer solo las IPs únicas del resultado
cat hosts_abiertos.txt | grep "open" | awk '{print $4}' | sort -u > ips_unicas.txt

# Paso 3 — Nmap: análisis detallado de los hosts encontrados
nmap -sV -sC -iL ips_unicas.txt -oA resultado_nmap

# Alternativa — pasar directamente a Nmap con los puertos encontrados
sudo masscan 10.0.0.0/8 -p 80,443 --rate 50000 -oX masscan.xml
# Convertir XML de Masscan a lista para Nmap
python3 -c "
import xml.etree.ElementTree as ET
tree = ET.parse('masscan.xml')
for host in tree.findall('.//host'):
    ip = host.find('.//address').get('addr')
    print(ip)
" > hosts.txt
nmap -A -iL hosts.txt
```

---

## Opciones avanzadas

```bash
# Tiempo de espera entre el último paquete enviado y el final del escaneo
# Por defecto es 10 segundos — aumentar en redes lentas
sudo masscan 192.168.1.0/24 -p 80 --wait 30

# Número de reintentos por puerto
sudo masscan 192.168.1.0/24 -p 80 --retries 2

# Rotar la dirección IP de origen (requiere múltiples IPs en la interfaz)
sudo masscan 192.168.1.0/24 -p 80 --source-ip 192.168.1.100-192.168.1.110

# Escaneo en modo sigiloso — TTL bajo para evitar IDS
sudo masscan 192.168.1.0/24 -p 80 --ttl 3

# Deshabilitar la detección de adaptador de red (modo offline para pruebas)
sudo masscan 192.168.1.0/24 -p 80 --offline

# Ver estadísticas detalladas durante el escaneo
sudo masscan 192.168.1.0/24 -p 80 --status-file /dev/stderr

# Semilla aleatoria para el orden de escaneo (reproducibilidad)
sudo masscan 192.168.1.0/24 -p 80 --seed 12345

# Mostrar todos los paquetes enviados y recibidos (debug)
sudo masscan 192.168.1.0/24 -p 80 --packet-trace
```

---

## Uso en pentesting — casos prácticos

### Descubrimiento inicial de una red corporativa

```bash
# Escaneo completo de todos los puertos en una red interna
sudo masscan 10.0.0.0/8 -p 0-65535 --rate 50000 -oL todos_puertos.txt

# Escaneo de puertos críticos en red corporativa
sudo masscan 10.0.0.0/8 -p 21,22,23,25,53,80,135,139,443,445,1433,1521,3306,3389,5432,5900,6379,8080,8443,27017 --rate 100000 -oL puertos_criticos.txt
```

### Detectar servicios específicos

```bash
# Encontrar todos los servidores web
sudo masscan 10.0.0.0/8 -p 80,443,8080,8443,8888 --rate 50000 -oL webs.txt

# Encontrar todos los servicios de escritorio remoto
sudo masscan 10.0.0.0/8 -p 3389,5900,5901,5902 --rate 50000 -oL rdp_vnc.txt

# Encontrar bases de datos expuestas
sudo masscan 10.0.0.0/8 -p 1433,1521,3306,5432,27017,6379,9200 --rate 50000 -oL bbdd.txt

# Encontrar servicios SSH
sudo masscan 10.0.0.0/8 -p 22 --rate 50000 -oL ssh.txt

# Encontrar servidores SMB (Windows)
sudo masscan 10.0.0.0/8 -p 445 --rate 50000 -oL smb.txt
```

### Reconocimiento en bug bounty

```bash
# Escaneo de un rango de IPs de una empresa (siempre con autorización)
sudo masscan 203.0.113.0/24 -p 0-65535 --rate 1000 --banners -oX resultado.xml

# Escaneo de puertos no estándar donde suelen esconderse servicios
sudo masscan 203.0.113.0/24 -p 8000-9000,10000-11000 --rate 1000 -oL no_estandar.txt
```

---

## Comparativa con Nmap

| Característica | Masscan | Nmap |
|----------------|---------|------|
| Velocidad | ⭐⭐⭐⭐⭐ (millones pps) | ⭐⭐ (miles pps) |
| Detección de versiones | ❌ básica (banners) | ✅ completa (-sV) |
| Scripts NSE | ❌ no soporta | ✅ biblioteca completa |
| Precisión en redes lentas | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| Escaneo UDP | ✅ básico | ✅ completo |
| Detección de OS | ❌ | ✅ |
| Reanudar escaneos | ✅ | ❌ |
| Rangos enormes | ✅ ideal | ⭐⭐ lento |

---

## Opciones más usadas — resumen rápido

```bash
-p PUERTOS          # Puertos a escanear
--rate N            # Paquetes por segundo
--banners           # Capturar banners de servicios
-oX / -oJ / -oL     # Formato de salida (XML / JSON / lista)
--interface         # Interfaz de red
--exclude           # Excluir IPs o rangos
-iL FICHERO         # Leer targets desde fichero
--wait N            # Segundos de espera al final
--retries N         # Reintentos por puerto
--ping              # Escaneo ICMP
--resume            # Reanudar escaneo pausado
-c FICHERO          # Usar fichero de configuración
--echo              # Generar plantilla de configuración
--packet-trace      # Modo debug — mostrar paquetes
```

---

## Notas importantes

- Masscan **no detecta versiones de servicios** — solo confirma que un puerto está abierto. Para versiones usar Nmap con `-sV`.
- A velocidades muy altas puede generar **falsos negativos** — puertos abiertos que no detecta porque los paquetes se pierden.
- Masscan **requiere root** porque crea sockets raw que omiten la pila TCP del sistema operativo.
- En redes con **IDS/IPS** activos, velocidades altas dispararán alertas inmediatamente.
- Siempre usar `--exclude` para evitar escanear rangos de broadcast, multicast o IPs de infraestructura crítica.
- En CTFs y laboratorios con ASLR o configuraciones especiales, combinar siempre con Nmap para verificar resultados.
