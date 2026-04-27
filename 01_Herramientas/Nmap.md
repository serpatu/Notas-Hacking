# Nmap

## Descripción

Nmap (Network Mapper) es la herramienta de escaneo de redes más utilizada en ciberseguridad. Permite descubrir hosts activos, puertos abiertos, servicios en ejecución, versiones de software y sistemas operativos. Es el primer paso en cualquier auditoría de seguridad.

---

## Sintaxis general

```shell
nmap [opciones] TARGET
```

El TARGET puede ser:
- Una IP: `192.168.56.101`
- Un rango: `192.168.56.1-254`
- Una subred: `192.168.56.0/24`
- Un dominio: `ejemplo.com`
---
## Comando muy Usado

```
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn <IP>
```

---

## Fases de un escaneo Nmap

Nmap trabaja en fases secuenciales. Entenderlas te ayuda a elegir las opciones correctas:

```
1. Resolución DNS         → Convierte dominios a IPs
2. Host discovery         → ¿Está el host activo?
3. Port scanning          → ¿Qué puertos están abiertos?
4. Version detection      → ¿Qué servicio y versión corre en cada puerto?
5. OS detection           → ¿Qué sistema operativo tiene?
6. Script scanning        → Scripts NSE para información adicional
7. Traceroute             → Ruta hasta el host
```

---

## Estados de un puerto

| Estado | Significado |
|--------|-------------|
| **open** | El puerto acepta conexiones — hay un servicio escuchando |
| **closed** | El puerto responde pero no hay servicio |
| **filtered** | Un firewall bloquea la respuesta — no se sabe si está abierto |
| **unfiltered** | Accesible pero no se puede determinar si abierto o cerrado |
| **open\|filtered** | No se puede distinguir entre abierto y filtrado |

---

## Opciones de escaneo de puertos

### Especificar puertos

```shell
nmap -p 22 TARGET                  # Solo puerto 22
nmap -p 22,80,443 TARGET           # Puertos específicos
nmap -p 1-1000 TARGET              # Rango de puertos
nmap -p- TARGET                    # TODOS los puertos (1-65535)
nmap --top-ports 100 TARGET        # Los 100 puertos más comunes
nmap --top-ports 1000 TARGET       # Los 1000 puertos más comunes (por defecto)
```

### Tipos de escaneo

```shell
nmap -sS TARGET    # SYN scan (stealth) — el más usado, requiere root ⭐
nmap -sT TARGET    # TCP Connect scan — sin root, más ruidoso
nmap -sU TARGET    # UDP scan — para servicios UDP (DNS, SNMP, DHCP)
nmap -sN TARGET    # NULL scan — para evadir algunos firewalls
nmap -sF TARGET    # FIN scan — para evadir algunos firewalls
nmap -sX TARGET    # Xmas scan — para evadir algunos firewalls
```

> **SYN scan (-sS):** Envía un paquete SYN y espera SYN-ACK (abierto) o RST (cerrado). No completa el handshake TCP, por eso es más sigiloso. Requiere privilegios de root.

> **TCP Connect (-sT):** Completa el handshake TCP completo. No requiere root pero deja más rastro en logs.

---

## Detección de versiones y OS

```shell
nmap -sV TARGET              # Detectar versión de servicios ⭐
nmap -sV --version-intensity 9 TARGET   # Máxima intensidad de detección
nmap -O TARGET               # Detectar sistema operativo (requiere root)
nmap -A TARGET               # Todo a la vez: -sV -O -sC --traceroute ⭐
```

---

## Scripts NSE (Nmap Scripting Engine)

Los scripts NSE amplían enormemente las capacidades de Nmap. Están en `/usr/share/nmap/scripts/`.

```shell
nmap -sC TARGET              # Scripts por defecto (safe) ⭐
nmap --script=banner TARGET  # Script específico
nmap --script=smb-vuln* TARGET              # Todos los scripts que empiecen por smb-vuln
nmap --script=http-title,http-headers TARGET # Varios scripts
```

### Scripts más útiles por servicio

```shell
# SMB
nmap -p 445 --script smb-vuln* TARGET          # Vulnerabilidades SMB
nmap -p 445 --script smb-enum-shares TARGET    # Listar shares
nmap -p 445 --script smb-enum-users TARGET     # Listar usuarios

# HTTP
nmap -p 80 --script http-title TARGET          # Título de la web
nmap -p 80 --script http-robots.txt TARGET     # Archivo robots.txt
nmap -p 80 --script http-shellshock TARGET     # Shellshock

# FTP
nmap -p 21 --script ftp-anon TARGET            # Acceso anónimo
nmap -p 21 --script ftp-vuln* TARGET           # Vulnerabilidades FTP

# SSH
nmap -p 22 --script ssh-auth-methods TARGET    # Métodos de autenticación
nmap -p 22 --script ssh-hostkey TARGET         # Clave del host

# MySQL
nmap -p 3306 --script mysql-info TARGET        # Info del servidor
nmap -p 3306 --script mysql-empty-password TARGET  # Root sin contraseña
```

---

## Velocidad y rendimiento

```shell
nmap -T0 TARGET    # Paranoid — muy lento, máximo sigilo
nmap -T1 TARGET    # Sneaky — lento
nmap -T2 TARGET    # Polite — moderado
nmap -T3 TARGET    # Normal — por defecto
nmap -T4 TARGET    # Aggressive — rápido ⭐ (usar en laboratorio)
nmap -T5 TARGET    # Insane — muy rápido, puede perder resultados

# Control manual de velocidad
nmap --min-rate 5000 TARGET    # Mínimo 5000 paquetes/segundo ⭐
nmap --max-rate 1000 TARGET    # Máximo 1000 paquetes/segundo
```

> En máquinas de práctica usa siempre `-T4` o `--min-rate 5000` para ir rápido. En entornos reales usa `-T2` o `-T3` para no saturar la red ni alertar al IDS.

---

## Host Discovery (detectar hosts activos)

```shell
nmap -sn 192.168.56.0/24       # Ping scan — solo descubrir hosts activos, sin escanear puertos
nmap -Pn TARGET                # Saltar ping — tratar el host como activo aunque no responda
nmap -PS22,80,443 TARGET       # SYN ping en puertos específicos
nmap -PE TARGET                # ICMP echo ping
```

> `-Pn` es muy útil cuando el host tiene el ping bloqueado por firewall. En máquinas CTF úsalo si nmap no encuentra nada con el escaneo normal.

---

## Output — Guardar resultados

```shell
nmap -oN resultado.txt TARGET      # Formato normal (legible) ⭐
nmap -oX resultado.xml TARGET      # Formato XML
nmap -oG resultado.gnmap TARGET    # Formato grepable
nmap -oA resultado TARGET          # Los tres formatos a la vez
```

---

## Evasión de firewalls

```shell
nmap -f TARGET                 # Fragmentar paquetes
nmap -D RND:10 TARGET          # Decoy — mezclar con IPs falsas
nmap --source-port 53 TARGET   # Usar puerto 53 como origen (DNS)
nmap -sI zombie TARGET         # Idle scan — usar host zombie
nmap --data-length 25 TARGET   # Añadir datos aleatorios a los paquetes
```

---

## Combinaciones más usadas en CTF/pentesting

### Escaneo inicial rápido — descubrir puertos abiertos

```shell
nmap -p- --min-rate 5000 TARGET
```

### Escaneo detallado de puertos encontrados

```shell
nmap -p 22,80,445 -sV -sC TARGET
```

### Escaneo completo todo en uno

```shell
nmap -p- -sV -sC -O --min-rate 5000 TARGET
```

### Buscar vulnerabilidades específicas

```shell
nmap -p 445 --script smb-vuln* TARGET
nmap -p 80 --script vuln TARGET
```

### Descubrir hosts en una red

```shell
nmap -sn 192.168.56.0/24
```

---

## Flujo recomendado en una máquina nueva

```
# Paso 1 — Escaneo rápido de todos los puertos
nmap -p- --min-rate 5000 TARGET_IP

# Paso 2 — Anotar los puertos abiertos (ej: 22,80,445)
# Paso 3 — Escaneo detallado solo de esos puertos
nmap -p 22,80,445 -sV -sC TARGET_IP

# Paso 4 — Si hay servicios interesantes, lanzar scripts específicos
nmap -p 445 --script smb-vuln* TARGET_IP
```

> Hacer el escaneo en dos pasos es más eficiente: el `-p-` rápido descubre todos los puertos, y luego el `-sV -sC` profundiza solo donde merece la pena.

---

## Interpretación de resultados

```
PORT     STATE  SERVICE  VERSION
22/tcp   open   ssh      OpenSSH 7.9 (protocol 2.0)
80/tcp   open   http     Apache httpd 2.4.38
445/tcp  open   smb      Samba 4.9.5
3306/tcp closed mysql
8080/tcp filtered http-proxy
```

| Campo | Qué hacer |
|-------|-----------|
| Puerto `open` + servicio conocido | Buscar en metodologías por puerto |
| Versión específica | `searchsploit nombre versión` |
| Puerto `filtered` | Probar con `-Pn` o desde dentro de la red |
| Puerto inesperado | Buscar en Google qué servicio usa ese puerto |
