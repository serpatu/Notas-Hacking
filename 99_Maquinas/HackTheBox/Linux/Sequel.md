# Máquina: Sequel

**IP:** 10.129.242.191
**OS:** Linux
**Dificultad:** Very Easy (Starting Point)
**Servicio Clave:** MySQL / MariaDB
**Fecha:** 2025-11-21
**Skills:** #MySQL #Database #Misconfiguration #ClientTools

---
## 1. Reconocimiento
Escaneo de puertos para identificar servicios expuestos:

```bash
nmap -p- -sV --min-rate 5000 10.129.242.191
```
## 2. Conexión y Explotación
Intentamos conectarnos al servicio MySQL remoto. Como es una configuración por defecto común en entornos de prueba, probamos el usuario **root** sin contraseña.

**Comando de conexión:**
- `-h`: Especifica el Host (la IP objetivo).
- `-u`: Especifica el Usuario (`root`).
- `--skip-ssl`: **Crucial en HTB.** Evita errores de certificado en servidores antiguos.

```bash
mysql -h 10.129.242.191 -u root --skip-ssl
```
## 3. Enumeración Interna (Post-Explotación)
Una vez obtenida la shell de MariaDB (`MariaDB [(none)]>`), enumeramos el contenido para encontrar datos sensibles.

**1. Identificar Bases de Datos:**
Listamos las bases de datos existentes.
```sql
SHOW DATABASES;
```


