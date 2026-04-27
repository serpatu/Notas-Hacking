# Responder

**Categoría:** Network Spoofing / Man-in-the-Middle
**Función:** Herramienta que "envenena" la red (LLMNR, NBT-NS y MDNS) para engañar a las máquinas Windows y hacerse pasar por un servidor legítimo.
**Objetivo:** Capturar credenciales (Hashes NTLMv1/v2) cuando una víctima intenta autenticarse contra nosotros.
**Etiquetas:** #ntlm #windows #spoofing #red

## Uso Principal

### Escuchar tráfico en una interfaz
El comando básico para iniciar la herramienta y ver si "pescamos" algo.

```bash
sudo responder -I <interfaz>
```
## Configuración Avanzada y Trucos

### 1. El Archivo de Configuración
Responder es muy "codicioso" y levanta servidores en muchos puertos (HTTP 80, SMB 445, etc.). Si intentas usar otras herramientas que necesiten esos puertos mientras Responder funciona, fallarán.

**Ruta del archivo:**
`/etc/responder/Responder.conf` (o `/usr/share/responder/Responder.conf`)

**Uso común:**
Editar este archivo para poner en `Off` los servidores que no queramos usar (por ejemplo, apagar el servidor HTTP para usar el nuestro propio).

```ini
[Responder Core]
; Servers to start
SQL = On
SMB = On    <-- Cambiar a Off si queremos usar smbserver.py
Kerberos = On
FTP = On
POP = On
SMTP = On
IMAP = On
HTTP = On   <-- Cambiar a Off si queremos levantar un python http.server
HTTPS = On
DNS = On
```
### 2. Modo análisis

```
sudo responder -I <interfaz> -A
```