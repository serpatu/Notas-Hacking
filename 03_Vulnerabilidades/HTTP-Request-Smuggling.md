# HTTP Request Smuggling (CL.TE / TE.CL / TE.TE)

## Descripción

Técnica que explota una **discrepancia entre un proxy delantero y un backend** sobre dónde termina una petición HTTP, cuando ambos reciben las cabeceras `Content-Length` y `Transfer-Encoding` a la vez y no se ponen de acuerdo sobre cuál usar. El resultado: bytes que uno de los dos interpreta como "sobrantes" se cuelan como el principio de la **siguiente** petición en la misma conexión reutilizada — permitiendo saltarse controles (como autenticación) que solo aplica el proxy delantero.

---
## Base: Content-Length vs Transfer-Encoding

Con HTTP/1.1 keep-alive, varias peticiones comparten la misma conexión TCP. El servidor necesita saber exactamente dónde termina el cuerpo de cada una:

- **`Content-Length: N`** — el cuerpo mide exactamente N bytes.
- **`Transfer-Encoding: chunked`** — el cuerpo llega en trozos, cada uno con su tamaño en hexadecimal, terminando con un trozo `0`:
```
5\r\n
Hola!\r\n
0\r\n
\r\n
```

El RFC dice que si llegan ambas cabeceras, hay que usar `Transfer-Encoding` e ignorar `Content-Length` — pero no todas las implementaciones lo siguen igual, y con **dos servidores en cadena** (proxy + backend) pueden no coincidir en cuál usar.

---
## Clasificación (nombre = proxy.backend)

- **CL.TE** — el proxy usa `Content-Length`, el backend usa `Transfer-Encoding`.
- **TE.CL** — el proxy usa `Transfer-Encoding`, el backend usa `Content-Length`.
- **TE.TE** — ambos dicen usar `Transfer-Encoding`, pero uno de los dos puede ser engañado para que la ignore (cabecera duplicada, valor mal escrito como `chunkedx`, espacio antes de los dos puntos...), comportándose de facto como uno de los dos casos anteriores.

---
## Detección por timing

`curl` no sirve — normaliza las cabeceras y no deja mandar conflictos a propósito. Hace falta mandar bytes en crudo por socket.

### CL.TE
```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
X
```
Si el proxy usa CL, reenvía solo 4 bytes (`1\r\nA`) y da la petición por completa. Si el backend usa TE, espera el resto del chunk (`\r\n` + `0\r\n\r\n`) que nunca llega → **timeout**.

### TE.CL
```
POST / HTTP/1.1
Host: TARGET
Content-Length: 30
Transfer-Encoding: chunked

0

```
Si el proxy usa TE, ve el chunk `0` (cuerpo vacío) y da la petición por completa — son solo 5 bytes reales. Si el backend usa CL, espera 30 bytes y solo llegaron 5 → **timeout**.

### Script de detección (Python, socket crudo)
```python
import socket, time

def send_raw(host, port, payload: bytes, timeout=8):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(timeout)
    s.connect((host, port))
    start = time.time()
    s.sendall(payload)
    try:
        data = s.recv(4096)
        print(f"[+] Respuesta en {time.time()-start:.2f}s:\n{data.decode(errors='replace')}")
    except socket.timeout:
        print(f"[!] TIMEOUT tras {time.time()-start:.2f}s -> posible smuggling")
    finally:
        s.close()
```

---
## Herramientas

- **smuggler.py** (`defparam/smuggler` en GitHub) — automatiza las variantes CL.TE / TE.CL / TE.TE.
- **Burp Suite + extensión "HTTP Request Smuggler"** (BApp Store, Albinowax) — usa la técnica de "single-packet attack" para evitar falsos positivos por buffering de red. En Repeater: pestaña **Raw**, y desactivar **"Update Content-Length header"** (si no, Burp "corrige" el conflicto que estás creando a propósito).

---
## Por qué puede fallar (falso negativo)

Versiones modernas de nginx **detectan y rechazan con 400 Bad Request** cualquier petición con `Content-Length` y `Transfer-Encoding` en conflicto, de forma inmediata (sin reenviarla a ningún backend). Si se obtiene un 400 instantáneo (no un timeout) en ambos tests, esa variante "básica" está bloqueada en la primera capa — no implica necesariamente que el sitio no sea vulnerable por otra vía (ofuscación TE.TE), pero sí descarta el ataque directo.

---
## Por qué importa el reenvío entre clientes

El proxy delantero mantiene un **pool de conexiones persistentes** hacia el backend, reutilizadas entre peticiones de clientes distintos. Si el proxy hace la comprobación de autenticación (p. ej. Basic Auth) pero el backend confía en una frontera distinta para el final de la petición, los bytes "colados" nunca vuelven a pasar por esa comprobación — llegan directos al backend como si fueran la siguiente petición legítima en la cola. En el peor caso, pueden terminar pegados delante de la petición de **otro usuario real** que reutilice esa misma conexión después.
