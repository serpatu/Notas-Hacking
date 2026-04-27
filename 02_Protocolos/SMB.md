# SMB — Server Message Block

## Descripción

SMB (Server Message Block) es el protocolo de red de Microsoft para compartir archivos, impresoras y otros recursos entre máquinas de una red. Es el protocolo fundamental en entornos Windows y también está disponible en Linux mediante su implementación open source **Samba**.

**Puertos por defecto:**
- **139** — NetBIOS Session Service (SMB sobre NetBIOS, versiones antiguas)
- **445** — SMB directo sobre TCP/IP (versiones modernas)

**Protocolo de transporte:** TCP
**Cifrado:** Opcional desde SMB 3.0

---

## Versiones de SMB

| Versión | Sistema | Notas de seguridad |
|---------|---------|-------------------|
| SMBv1 | Windows XP/2003 | **Obsoleto y peligroso** — EternalBlue, WannaCry |
| SMBv2 | Windows Vista/2008 | Mejoras de rendimiento |
| SMBv3 | Windows 8/2012 | Añade cifrado end-to-end |

> SMBv1 debe estar desactivado siempre. Si lo ves activo en un nmap, es señal de sistema sin parchear.

---

## Cómo funciona

SMB permite a los clientes:
- Acceder a archivos y directorios compartidos (**shares**)
- Imprimir en impresoras compartidas
- Acceder a recursos de red como si fueran locales
- Comunicación entre procesos (IPC$)

### Shares especiales (siempre presentes en Windows)

| Share | Descripción |
|-------|-------------|
| `C$` | Acceso al disco C (solo admins) |
| `ADMIN$` | Directorio de Windows (solo admins) |
| `IPC$` | Canal de comunicación entre procesos |

---

## Samba en Linux

Samba es la implementación open source de SMB para Linux. Permite que máquinas Linux participen en redes Windows compartiendo archivos e impresoras.

Configuración principal: `/etc/samba/smb.conf`

```ini
[share_name]
path = /ruta/al/directorio
browseable = yes
writable = yes
guest ok = yes          # Sin contraseña — inseguro
```

---

## Comandos básicos con smbclient

```shell
# Listar shares disponibles sin autenticación
smbclient -L //TARGET_IP -N

# Listar shares con credenciales
smbclient -L //TARGET_IP -U usuario%contraseña

# Conectar a un share
smbclient //TARGET_IP/nombre_share -N
smbclient //TARGET_IP/nombre_share -U usuario%contraseña

# Dentro de smbclient
ls              # Listar archivos
get archivo     # Descargar
put archivo     # Subir
mget *          # Descargar todo
```

---

## Autenticación en SMB

SMB usa el protocolo **NTLM** para autenticación:

1. El cliente solicita acceso
2. El servidor envía un **challenge** (número aleatorio)
3. El cliente cifra el challenge con el hash NTLM de la contraseña
4. El servidor verifica el resultado

Esto significa que con el **hash NTLM** de un usuario puedes autenticarte sin saber la contraseña en texto plano — técnica conocida como **Pass-the-Hash**.

---

## Vulnerabilidades históricas importantes

| CVE | Nombre | Versión afectada | Impacto |
|-----|--------|-----------------|---------|
| MS17-010 | EternalBlue | Windows 7/2008 R2 sin parchear | RCE sin autenticación |
| MS08-067 | — | Windows XP/2003 | RCE sin autenticación |
| CVE-2017-7494 | SambaCry | Samba < 4.6.4 | RCE sin autenticación |

---

## Herramientas relacionadas

| Herramienta | Uso |
|-------------|-----|
| `smbclient` | Cliente SMB de línea de comandos |
| `enum4linux` | Enumeración completa de SMB/Samba |
| `smbmap` | Mapear shares y permisos |
| `impacket-psexec` | Ejecución remota de comandos con credenciales |
| `impacket-smbclient` | Cliente SMB alternativo |
| `crackmapexec` | Enumeración y ataque automatizado |

---

## Debilidades de seguridad

- SMBv1 activo → vulnerable a EternalBlue
- Shares accesibles sin autenticación (guest)
- Shares con escritura permiten subir archivos maliciosos
- Credenciales débiles vulnerables a fuerza bruta
- Transmisión de hashes NTLM interceptable para ataques Pass-the-Hash
