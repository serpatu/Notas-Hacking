# Evil-WinRM

**Categoría:** Shell Remota / Post-Explotación Windows
**Función:** Shell interactiva para sistemas Windows con el servicio WinRM habilitado (Puertos 5985 HTTP / 5986 HTTPS).
**Requisito:** Credenciales válidas (Usuario+Password o Usuario+Hash NTLM).
**Lenguaje:** Ruby.
**Etiquetas:** #windows #shell #winrm #pth #lateral-movement

---

## 🚀 Conexión Básica

### Con Contraseña (Texto plano)
El estándar cuando has crackeado la clave.
```bash
evil-winrm -i <IP> -u <USUARIO> -p <CONTRASEÑA>
```