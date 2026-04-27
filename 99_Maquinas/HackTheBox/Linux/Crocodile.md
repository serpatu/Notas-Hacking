# Máquina: Crocodile

**IP:** 10.129.247.64
**OS:** Linux
**Dificultad:** Very Easy (Starting Point)
**Servicio Clave:** FTP
**Fecha:** 2025-11-21
**Skills:** #FTP #AnonymousLogin #InformationDisclosure #CredentialReuse #Web

---
## 1. Reconocimiento
**Escaneo de puertos y servicios:**
Lanzamos Nmap con scripts básicos (`-sC`) para enumerar versiones y vulnerabilidades comunes.

```bash
nmap -p- -sVC --min-rate 5000 10.129.247.64
```
## 2. Explotación FTP (Exfiltración de Datos)
Aprovechamos la vulnerabilidad de **Login Anónimo** detectada por Nmap para acceder al servidor de archivos y descargar la información sensible.

**1. Conexión:**
```bash
ftp 10.129.247.64
ftp> ls
ftp> get allowed.userlist
ftp> get allowed.userlist.passwd
ftp> exit
cat allowed.userlist
cat allowed.userlist.passwd
```
## 3. Enumeración Web y Acceso
Con las credenciales exfiltradas del FTP, pivotamos hacia el servicio web (Puerto 80) para buscar un panel de administración.

**1. Localización del Login:**
Navegamos a la web principal. Al no ver un botón de acceso evidente, probamos rutas estándar (Fuzzing manual o intuición).
* **Ruta identificada:** `http://10.129.247.64/login.php`

**2. Autenticación:**
Utilizamos la información robada:
* **Usuario:** Probamos `admin` (presente en `allowed.userlist` y el más común).
* **Contraseña:** La cadena de texto encontrada en `allowed.userlist.passwd`.

**Acción:**
Introducimos las credenciales en el formulario de `login.php` para intentar acceder al dashboard.