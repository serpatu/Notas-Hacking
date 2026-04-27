# FTP — File Transfer Protocol

## Descripción

FTP (File Transfer Protocol) es un protocolo estándar de red para la transferencia de archivos entre un cliente y un servidor. Fue diseñado en 1971 y es uno de los protocolos más antiguos de Internet.

**Puerto por defecto:** 21 (control) / 20 (datos activo)
**Protocolo de transporte:** TCP
**Cifrado:** Ninguno en FTP estándar. FTPS añade TLS/SSL.

---

## Cómo funciona

FTP usa **dos canales separados**:

- **Canal de control (puerto 21):** Para enviar comandos y recibir respuestas. Permanece abierto durante toda la sesión.
- **Canal de datos (puerto 20 o aleatorio):** Para la transferencia real de archivos. Se abre y cierra por cada transferencia.

### Modos de conexión

**Modo activo:** El servidor inicia la conexión de datos hacia el cliente. Puede tener problemas con firewalls del lado del cliente.

**Modo pasivo (PASV):** El cliente inicia ambas conexiones. Es el modo estándar hoy en día porque atraviesa mejor los firewalls.

---

## Tipos de acceso

**Acceso autenticado:** Requiere usuario y contraseña válidos del sistema.

**Acceso anónimo:** Permite conectarse con usuario `anonymous` y cualquier email como contraseña. Pensado para distribución pública de archivos, pero frecuentemente mal configurado.

---

## Comandos básicos del cliente FTP

```shell
# Conectar
ftp TARGET_IP

# Una vez conectado
ls          # Listar archivos
ls -la      # Listar incluyendo ocultos
cd dir      # Cambiar directorio
pwd         # Directorio actual
get file    # Descargar archivo
put file    # Subir archivo
mget *      # Descargar todos los archivos
mput *      # Subir todos los archivos
binary      # Modo binario (para archivos no texto)
ascii       # Modo texto
bye         # Salir
```

---

## Implementaciones más comunes

| Software | Sistema | Notas |
|----------|---------|-------|
| vsftpd | Linux | Muy común en Ubuntu/Debian |
| ProFTPD | Linux | Configurable y extensible |
| Pure-FTPd | Linux | Ligero y seguro |
| FileZilla Server | Windows | Con interfaz gráfica |
| IIS FTP | Windows | Integrado en IIS |

---

## Variantes seguras

| Protocolo | Descripción | Puerto |
|-----------|-------------|--------|
| **FTPS** | FTP sobre TLS/SSL | 990 |
| **SFTP** | FTP sobre SSH (protocolo diferente) | 22 |

> SFTP no tiene nada que ver con FTP internamente — es un subsistema de SSH. Solo comparte el nombre.

---

## Debilidades de seguridad

- Transmite credenciales y datos **en texto plano** — vulnerable a sniffing
- El acceso anónimo mal configurado expone archivos sensibles
- Versiones antiguas tienen backdoors y vulnerabilidades conocidas (vsftpd 2.3.4)
- Permite subida de archivos si los permisos están mal configurados

---

## Ejemplo real — Máquina Metasploitable 2

Metasploitable 2 tiene vsftpd 2.3.4 en el puerto 21, que contiene un backdoor que abre una shell en el puerto 6200 cuando el usuario de login termina en `:)`.
