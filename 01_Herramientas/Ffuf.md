# ffuf — Fuzz Faster U Fool

## Descripción

ffuf es una herramienta de fuzzing web muy rápida y flexible. Es similar a Gobuster pero más potente en cuanto a opciones de filtrado y velocidad. Se usa principalmente para descubrir directorios, archivos, parámetros y valores en aplicaciones web.

**Ventaja principal sobre Gobuster:** Permite filtrar respuestas por tamaño, palabras, líneas y código de estado simultáneamente, lo que reduce enormemente los falsos positivos.

---

## Instalación

Viene preinstalada en Kali Linux:

```shell
ffuf -h
```

---

## Sintaxis general

```shell
ffuf -u URL_CON_FUZZ -w WORDLIST [opciones]
```

La palabra `FUZZ` en la URL es el marcador que ffuf sustituye por cada palabra de la wordlist.

---

## Casos de uso más comunes

### Descubrir directorios

```shell
ffuf -u http://TARGET_IP/FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200,301,302 \
  -t 100
```

### Descubrir archivos con extensiones

```shell
ffuf -u http://TARGET_IP/FUZZ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -e .php,.txt,.html,.bak \
  -mc 200 \
  -t 100
```

### Descubrir directorios ~usuario (Apache mod_userdir) 

```shell
# Con wordlist de directorios genéricos — NO de usuarios
ffuf -u "http://TARGET_IP/~FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200,301,302 \
  -t 100
```

### Descubrir archivos ocultos (empiezan por punto)

```shell
ffuf -u "http://TARGET_IP/directorio/.FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -e .txt,.php,.html \
  -mc 200 \
  -t 100
```

### Fuzzear parámetros GET

```shell
ffuf -u "http://TARGET_IP/page.php?FUZZ=valor" \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -mc 200 \
  -t 50
```

### Fuzzear valores de parámetros

```shell
ffuf -u "http://TARGET_IP/page.php?id=FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200 \
  -t 50
```

---

## Opciones más importantes

### Wordlist y URL
```shell
-u URL        # URL con FUZZ como marcador
-w WORDLIST   # Ruta a la wordlist
-e .php,.txt  # Extensiones a añadir a cada palabra
```

### Filtros — para eliminar falsos positivos 

```shell
-mc 200,301   # Mostrar solo estos códigos de estado (match codes)
-fc 404,403   # Filtrar estos códigos de estado (filter codes)
-fs 1234      # Filtrar respuestas de este tamaño en bytes (filter size)
-fw 20        # Filtrar respuestas con este número de palabras (filter words)
-fl 10        # Filtrar respuestas con este número de líneas (filter lines)
```

### Rendimiento
```shell
-t 100        # Número de hilos (por defecto 40)
-rate 500     # Máximo de peticiones por segundo
```

### Output
```shell
-o resultado.txt    # Guardar resultados en archivo
-of json            # Formato de salida (json, csv, md, html)
-v                  # Modo verbose
-c                  # Colorear la salida
```

### Otros
```shell
-ic           # Ignorar comentarios en la wordlist
-s            # Silent mode — solo muestra resultados
-r            # Seguir redirects
-H "Header: valor"   # Añadir cabecera HTTP personalizada
-b "cookie=valor"    # Añadir cookie
```

---

## Cómo eliminar falsos positivos

El problema más común con ffuf es que devuelve demasiados resultados. La solución es filtrar por tamaño de respuesta:

**Paso 1 — Haz una petición a una ruta que no existe y anota el tamaño:**
```shell
curl -s http://TARGET_IP/ruta_que_no_existe_12345 | wc -c
# Supón que devuelve 1234 bytes
```

**Paso 2 — Filtra ese tamaño en ffuf:**
```shell
ffuf -u http://TARGET_IP/FUZZ \
  -w wordlist.txt \
  -fs 1234 \
  -t 100
```

---

## ffuf vs Gobuster — cuándo usar cada uno

| Situación | Herramienta recomendada |
|-----------|------------------------|
| Enumeración básica de directorios | Gobuster (más simple) |
| Necesitas filtrar por tamaño/palabras | ffuf ⭐ |
| Fuzzear parámetros GET/POST | ffuf ⭐ |
| Buscar archivos ocultos con `.FUZZ` | ffuf ⭐ |
| Directorios `~FUZZ` | ffuf ⭐ |
| Entorno CTF con muchos falsos positivos | ffuf ⭐ |

---

## Wordlists recomendadas por caso

| Caso | Wordlist |
|------|----------|
| Directorios generales | `dirbuster/directory-list-2.3-medium.txt` |
| Archivos web comunes | `seclists/Discovery/Web-Content/common.txt` |
| Parámetros | `seclists/Discovery/Web-Content/burp-parameter-names.txt` |
| Usuarios | `seclists/Usernames/Names/names.txt` |
| Contraseñas | `rockyou.txt` |

---

## Ejemplo real — Máquina Empire: LupinOne

El directorio `/~secret` se encontró usando ffuf con una wordlist de directorios genéricos:

```shell
ffuf -u "http://192.168.0.41/~FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -mc 200,301,302 \
  -t 100
# Resultado: /~secret [Status: 200]
```

Luego se buscaron archivos ocultos dentro de ese directorio:

```shell
ffuf -u "http://192.168.0.41/~secret/.FUZZ" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -e .txt \
  -mc 200 \
  -t 100
# Resultado: /.mysecret.txt [Status: 200] — clave SSH privada en base58
```

> **Lección aprendida:** Cuando fuzzes `~FUZZ` usa wordlists de directorios, no de usuarios. El nombre puede ser cualquier palabra.
