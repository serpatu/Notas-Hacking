# WinRM (Windows Remote Management)

**Puertos por defecto:**
- **5985** (HTTP)
- **5986** (HTTPS)

**Descripción:**
Es un protocolo de Microsoft basado en SOAP que permite la administración remota de servidores Windows. Es el equivalente a SSH en el mundo Linux, permitiendo ejecutar comandos de PowerShell remotamente.

## Herramientas de Pentesting

### Evil-WinRM
La herramienta por excelencia en Kali Linux para explotar este servicio si tenemos credenciales.

**Instalación:**
```bash
sudo apt install evil-winrm
```