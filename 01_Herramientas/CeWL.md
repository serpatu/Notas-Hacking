# CeWL — Custom Word List Generator

## Descripción

CeWL (Custom Word List Generator) es una herramienta que **rastrea un sitio web y extrae todas las palabras del contenido** para crear una wordlist personalizada. La lógica detrás es simple pero poderosa: las personas tienden a usar contraseñas relacionadas con su entorno — si administran un sitio web sobre seguridad, su contraseña probablemente contiene palabras de ese sitio.

**Cuándo usarla:** Cuando encuentres un sitio web con contenido propio y necesites hacer fuerza bruta. Especialmente útil cuando rockyou.txt y otras wordlists genéricas no funcionan.

---

## Instalación

Viene preinstalada en Kali Linux:

```shell
cewl --help
```

---

## Uso básico

```shell
# Sintaxis básica
cewl -d PROFUNDIDAD -m LONGITUD_MINIMA -w ARCHIVO_SALIDA URL
```

```shell
# Ejemplo básico
cewl -d 2 -m 5 -w wordlist.txt http://TARGET/
```

---

## Opciones más importantes

### Profundidad y longitud

```shell
-d N          # Profundidad del rastreo (cuántos niveles de links seguir)
              # -d 1 → solo la página principal
              # -d 2 → página principal + páginas enlazadas
              # -d 3 → un nivel más profundo (recomendado)
              
-m N          # Longitud mínima de palabra
              # -m 3 → palabras de 3+ caracteres
              # -m 5 → palabras de 5+ caracteres (menos ruido)
```

### Modificadores de palabras

```shell
--lowercase          # Convertir todas las palabras a minúsculas
                     # Muy útil — las contraseñas suelen ser en minúsculas

--with-numbers       # Incluir palabras que contengan números
                     # Útil para contraseñas como "admin123"

--count              # Mostrar cuántas veces aparece cada palabra
                     # Las palabras más frecuentes son más probables como contraseña
```

### Extracción adicional

```shell
-e               # Extraer emails encontrados en la web
                 # Los emails revelan nombres de usuarios potenciales

-a               # Extraer metadatos de documentos (PDFs, Word, etc.)
                 # El campo "Autor" puede ser un usuario válido del sistema
```

### Autenticación

```shell
--auth-type basic          # Autenticación HTTP Basic
--auth-user USER           # Usuario para autenticación
--auth-pass PASS           # Contraseña para autenticación
```

---

## Comandos más usados en CTFs

### Comando básico recomendado

```shell
cewl -d 3 -m 3 --lowercase -w wordlist.txt http://TARGET/
```

### Versión más completa

```shell
cewl -d 5 -m 3 --lowercase --with-numbers -w wordlist.txt http://TARGET/
```

### Con extracción de emails y metadatos

```shell
cewl -d 3 -m 3 --lowercase -e -a -w wordlist.txt http://TARGET/
```

### Combinar múltiples wordlists

```shell
cewl -d 3 -m 3 http://TARGET/ -w cewl1.txt
cewl -d 3 -m 3 --with-numbers http://TARGET/ -w cewl2.txt
cat cewl1.txt cewl2.txt | sort -u > wordlist_final.txt
```

---

## Por qué cewl funciona mejor que rockyou en CTFs

En CTFs las contraseñas están diseñadas para ser encontradas con las pistas dadas. Si la Flag 1 dice "maybe you just need to be cewl", significa directamente que hay que usar CeWL. Las contraseñas están elegidas del contenido del sitio.

En entornos reales también funciona bien porque:
- Los administradores de sistemas usan contraseñas relacionadas con su trabajo
- El contenido del sitio refleja el vocabulario del negocio
- Es más difícil de detectar que ataques con wordlists genéricas

---

## Limitaciones

- No funciona bien con SPAs (Single Page Applications) que cargan contenido con JavaScript
- El contenido mínimo genera pocas palabras — combinar con wordlists genéricas si no funciona
- Solo extrae palabras visibles en el HTML, no contenido dinámico

---

## Flujo de trabajo con cewl

```
1. Identificar el sitio web objetivo
         ↓
2. Navegar manualmente para entender el contenido
         ↓
3. cewl -d 3 -m 3 --lowercase -w wordlist.txt http://TARGET/
         ↓
4. wc -l wordlist.txt → verificar cuántas palabras generó
         ↓
5. Usar la wordlist con la herramienta de fuerza bruta apropiada:
   - WordPress → WPScan
   - SSH → Hydra
   - FTP → Hydra
   - Web login → Hydra / ffuf
         ↓
6. Si no funciona → ampliar con --with-numbers, más profundidad
         ↓
7. Si sigue sin funcionar → combinar con fasttrack.txt o rockyou.txt
```

---

## Ejemplo real — Máquina DC-2

La Flag 1 contenía la pista: *"maybe you just need to be cewl"*

```shell
cewl -d 3 -m 3 --lowercase -w palabras_cewl.txt http://dc-2/
```

Generó ~238 palabras del contenido Lorem ipsum de la web. La contraseña `adipiscing` era una palabra del texto de la web, y `parturient` también. Ambas aparecían en el contenido de las páginas del sitio.

```shell
wpscan --url http://dc-2 --usernames jerry,tom --passwords palabras_cewl.txt
# Resultado: jerry/adipiscing + tom/parturient
```
