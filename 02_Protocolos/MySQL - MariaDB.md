
# MySQL / MariaDB

**Puerto por defecto:** 3306
**Descripción:** Sistema de gestión de bases de datos relacional.

## Conexión desde Terminal
Para conectarnos a una base de datos remota usamos el cliente `mysql`.

**Comando básico:**
```bash
mysql -h <IP_OBJETIVO> -u <USUARIO> -p
```
- `-h`: Host (IP).
- `-u`: Usuario (ej: root).
- `-p`: Pedir contraseña (si no se pone, intenta entrar sin ella).