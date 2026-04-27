# SSH — Secure Shell

## Descripción

SSH (Secure Shell) es el protocolo estándar para acceso remoto seguro a sistemas Unix/Linux. Reemplazó a Telnet y rlogin al añadir cifrado completo de la comunicación. Fue diseñado en 1995.

**Puerto por defecto:** 22
**Protocolo de transporte:** TCP
**Cifrado:** Sí — toda la comunicación va cifrada

---

## Cómo funciona

SSH establece un canal cifrado entre cliente y servidor mediante criptografía asimétrica:

1. El cliente se conecta al servidor en el puerto 22
2. El servidor presenta su **clave pública** (host key)
3. El cliente la verifica contra `~/.ssh/known_hosts`
4. Se negocia el cifrado simétrico para la sesión
5. El usuario se autentica (por contraseña o clave)

---

## Métodos de autenticación

### Por contraseña
El método más simple. El usuario introduce su contraseña, que viaja cifrada.

```shell
ssh usuario@TARGET_IP
ssh -p 2222 usuario@TARGET_IP    # Puerto alternativo
```

### Por clave pública (más seguro)
Se genera un par de claves: privada (en el cliente) y pública (en el servidor en `~/.ssh/authorized_keys`).

```shell
# Generar par de claves en Kali
ssh-keygen -t rsa -b 4096 -f ~/.ssh/mi_clave

# Copiar clave pública al servidor
ssh-copy-id -i ~/.ssh/mi_clave.pub usuario@TARGET_IP

# Conectar con la clave privada
ssh -i ~/.ssh/mi_clave usuario@TARGET_IP
```

---

## Archivos importantes

| Archivo | Ubicación | Descripción |
|---------|-----------|-------------|
| `id_rsa` | `~/.ssh/id_rsa` | Clave privada del usuario |
| `id_rsa.pub` | `~/.ssh/id_rsa.pub` | Clave pública del usuario |
| `authorized_keys` | `~/.ssh/authorized_keys` | Claves públicas autorizadas para conectar |
| `known_hosts` | `~/.ssh/known_hosts` | Servidores conocidos y sus claves |
| `sshd_config` | `/etc/ssh/sshd_config` | Configuración del servidor SSH |

---

## Funcionalidades avanzadas

### Port Forwarding — acceder a servicios internos

```shell
# Local: acceder al puerto 8080 interno de la víctima desde Kali
ssh -L 8080:127.0.0.1:8080 usuario@TARGET_IP

# Dinámico: crear un proxy SOCKS para toda la red interna
ssh -D 1080 usuario@TARGET_IP
```

### Transferencia de archivos con SCP

```shell
# Subir archivo a la víctima
scp archivo.txt usuario@TARGET_IP:/tmp/

# Descargar archivo de la víctima
scp usuario@TARGET_IP:/etc/passwd ./passwd_remoto
```

### Túnel inverso — útil en post-explotación

```shell
# Desde la víctima: exponer su puerto 80 en el puerto 8080 de Kali
ssh -R 8080:127.0.0.1:80 kali@KALI_IP
```

---

## Configuración del servidor (sshd_config)

Opciones relevantes para pentesting:

```
PermitRootLogin yes/no          # Si root puede conectarse directamente
PasswordAuthentication yes/no   # Si se permite autenticación por contraseña
PubkeyAuthentication yes/no     # Si se permite autenticación por clave
Port 22                         # Puerto de escucha
```

---

## Debilidades de seguridad

- Versiones antiguas de OpenSSH tienen vulnerabilidades de enumeración de usuarios
- Si `PermitRootLogin yes` está activo, root puede atacarse por fuerza bruta
- Una clave privada `id_rsa` encontrada en el sistema permite acceso sin contraseña
- Contraseñas débiles son vulnerables a fuerza bruta con Hydra

---

## Implementaciones más comunes

| Software | Sistema |
|----------|---------|
| OpenSSH | Linux/Mac (estándar) |
| Dropbear | Dispositivos embebidos |
| PuTTY | Cliente Windows |
| OpenSSH para Windows | Windows 10+ |
