# TikiWiki

## Descripción

TikiWiki es una **plataforma de gestión de contenidos (CMS) y wiki** de código abierto. En entornos vulnerables como Metasploitable 2 se encuentra en versiones muy antiguas con vulnerabilidades conocidas y sin parchear.

**Puerto por defecto:** 80 (HTTP) / 443 (HTTPS)
**Ruta típica:** `/tikiwiki`
**Tecnología:** PHP + MySQL

---

## Identificación

Al acceder a la instalación, la versión aparece en el **pie de página**:

```
This is TikiWiki X.X.X -NombreVersion- © 2002-XXXX by the Tiki community
```

También se puede identificar con Gobuster buscando la ruta `/tikiwiki`.

---

## Vulnerabilidades conocidas

### SQLi en parámetro sort_mode (v1.9.5)

**CVE/Referencia:** Exploit-DB 2701
**Versión afectada:** TikiWiki 1.9.5 -Sirius-

El parámetro `sort_mode` se inserta directamente en la cláusula `ORDER BY` de MySQL sin sanitizar, provocando un error que revela información de la base de datos.

**Rutas vulnerables:**

```
/tiki-listpages.php?offset=0&sort_mode=
/tiki-lastchanges.php?days=1&offset=0&sort_mode=
/tiki-list_users.php?sort_mode=
/tiki-forums.php?sort_mode=
/tiki-list_blogs.php?sort_mode=
```

**Verificación manual** — visitar en el navegador:

```
http://TARGET/tikiwiki/tiki-listpages.php?offset=0&sort_mode=AAAA
```

Si devuelve un error MySQL como este, la vulnerabilidad existe:

```
Unknown column '' in 'order clause'
```

**Formato válido del parámetro:** TikiWiki espera el formato `columna_asc` o `columna_desc`. Ejemplo de valor legítimo:

```
sort_mode=pageName_asc
```

---

## Notas importantes

- La inyección está en la cláusula `ORDER BY`, no en `WHERE`. Esto hace que SQLMap tenga dificultades para detectarla automáticamente.
- Algunas instalaciones tienen validación adicional que bloquea payloads como `EXTRACTVALUE`, mostrando `Invalid variable value`.
- Muchas features como blogs, foros o shoutbox pueden estar desactivadas, devolviendo `This feature is disabled`.
- Las credenciales por defecto son `admin:admin` pero pueden estar cambiadas.

---

## Ejemplo real — Máquina Metasploitable 2

Encontramos TikiWiki 1.9.5 en `/tikiwiki`. Confirmamos la SQLi en `tiki-listpages.php` con el error MySQL pero la validación del parámetro impidió explotar la inyección directamente. Descartamos este vector y pivotamos a TWiki.
