# Enumeración y Ataque — SMTP (Puerto 25)

## Descripción

SMTP (Simple Mail Transfer Protocol) es el protocolo de envío de correo. En pentesting es útil principalmente para **enumerar usuarios válidos** del sistema, lo que luego permite ataques de fuerza bruta más dirigidos.

---

## Paso 1 — Enumeración con Nmap

```shell
nmap -p 25 -sV -sC TARGET_IP
```

---

## Paso 2 — Conexión manual y enumeración de usuarios

```shell
nc TARGET_IP 25
# o
telnet TARGET_IP 25
```

Una vez conectado, el servidor responde con su banner. Anota la versión.

### Comandos para enumerar usuarios

```shell
VRFY root          # Verifica si el usuario existe
VRFY admin
VRFY usuario

EXPN root          # Expande una lista de correo (a veces devuelve usuarios)

RCPT TO:<root>     # Otro método de verificación
```

Respuestas:
- `250` → usuario existe ✅
- `550` → usuario no existe ❌

---

## Paso 3 — Enumeración automática con Metasploit

```shell
use auxiliary/scanner/smtp/smtp_enum
set RHOSTS TARGET_IP
set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt
run
```

---

## Paso 4 — Buscar versión vulnerable

```shell
searchsploit smtp VERSION
searchsploit sendmail VERSION
searchsploit postfix VERSION
searchsploit exim VERSION
```

Exim tiene varios CVEs críticos de RCE. Si ves Exim, busca siempre exploits.

---

## Paso 5 — Open Relay (envío de correo sin autenticación)

Si el servidor permite enviar correos sin autenticarse (open relay), puedes usarlo para phishing:

```shell
nc TARGET_IP 25
EHLO kali
MAIL FROM:<atacante@evil.com>
RCPT TO:<victima@empresa.com>
DATA
Subject: Test
Mensaje de prueba.
.
QUIT
```

Si responde `250 OK` a todo, es un open relay.

---

## Checklist rápido

```
[ ] nmap -sV para ver versión y banner
[ ] Conectar con nc y probar VRFY con usuarios comunes
[ ] smtp_enum en Metasploit con wordlist de usuarios
[ ] searchsploit con versión exacta
[ ] Guardar lista de usuarios válidos para fuerza bruta en SSH/FTP
```
