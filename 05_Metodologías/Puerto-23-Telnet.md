# Enumeración y Ataque — Telnet (Puerto 23)

## Descripción

Telnet es un protocolo de acceso remoto antiguo que transmite todo en **texto plano sin cifrado**, incluidas las contraseñas. Ha sido reemplazado por SSH pero sigue apareciendo en máquinas antiguas y de práctica.

**Puerto por defecto:** 23

---

## Paso 1 — Conexión directa

```shell
telnet TARGET_IP
telnet TARGET_IP 23
nc TARGET_IP 23
```

El servidor muestra un banner de login. Anota el banner — puede revelar el sistema operativo o la versión.

---

## Paso 2 — Credenciales por defecto

```shell
# Prueba estas combinaciones
admin:admin
root:root
root:(vacía)
user:user
guest:guest
admin:password
```

---

## Paso 3 — Fuerza bruta

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt telnet://TARGET_IP
hydra -L usuarios.txt -P /usr/share/wordlists/rockyou.txt telnet://TARGET_IP
```

---

## Paso 4 — Buscar versión vulnerable

```shell
searchsploit telnet
searchsploit telnetd
```

---

## Paso 5 — Sniffing (si estás en la misma red)

Como Telnet no cifra, si estás en la misma red puedes capturar credenciales:

```shell
sudo tcpdump -i eth0 port 23 -A
# o con Wireshark filtrando por puerto 23
```

---

## Checklist rápido

```
[ ] telnet TARGET_IP → ver banner
[ ] Credenciales por defecto (root:root, admin:admin)
[ ] Fuerza bruta con Hydra
[ ] searchsploit telnetd con versión del banner
[ ] Si en misma red → tcpdump para capturar credenciales en texto plano
```
