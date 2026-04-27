# Esteganografía

## Descripción

La esteganografía es la técnica de **ocultar información dentro de archivos aparentemente normales** como imágenes, audio o vídeo. En CTFs es común encontrar datos ocultos en imágenes que contienen credenciales, claves SSH o flags.

> No confundir con criptografía — la criptografía cifra el mensaje, la esteganografía lo oculta dentro de otro archivo.

---

## Herramientas principales

### steghide — para JPEG/BMP/WAV 

```shell
# Extraer datos ocultos (pide passphrase — probar vacía primero)
steghide extract -sf imagen.jpg

# Ver información sin extraer
steghide info imagen.jpg

# Ocultar datos en una imagen
steghide embed -cf imagen.jpg -sf secreto.txt
```

> steghide NO soporta PNG — solo JPEG, BMP, WAV, AU.

### zsteg — para PNG 

```shell
# Instalación
sudo gem install zsteg

# Analizar imagen PNG
zsteg imagen.png
```

### binwalk — detectar archivos incrustados

```shell
# Ver qué hay dentro del archivo
binwalk imagen.jpg

# Extraer automáticamente
binwalk -e imagen.jpg
# Los archivos extraídos van a _imagen.jpg.extracted/
```

### exiftool — metadatos

```shell
# Ver metadatos de la imagen
exiftool imagen.jpg

# Buscar información sensible en metadatos
exiftool imagen.jpg | grep -i "comment\|description\|artist\|author"
```

### strings — texto legible

```shell
# Buscar texto legible dentro del archivo binario
strings imagen.jpg

# Buscar solo al final del archivo (donde suelen ocultarse datos)
strings imagen.jpg | tail -50
```

---

## Flujo de análisis en CTFs

```shell
# 1. Verificar tipo real del archivo (puede no coincidir con la extensión)
file imagen.jpg

# 2. Ver metadatos
exiftool imagen.jpg

# 3. Buscar texto legible
strings imagen.jpg

# 4. Buscar archivos incrustados
binwalk imagen.jpg

# 5. Intentar extracción con steghide (JPEG)
steghide extract -sf imagen.jpg
# → probar passphrase vacía, luego: el nombre de la máquina, admin, password...

# 6. Si es PNG, usar zsteg
zsteg imagen.png

# 7. Si binwalk encontró algo, extraer
binwalk -e imagen.jpg
ls _imagen.jpg.extracted/
```

---

## Pista importante — tipo de archivo vs extensión

Un archivo `.jpg` puede ser en realidad un PNG. Binwalk lo detecta:

```
DECIMAL    HEXADECIMAL    DESCRIPTION
0          0x0            PNG image, 630 x 630, 16-bit/color RGB
```

En este caso steghide no funcionará — usar zsteg en su lugar.

---

## Passphrase en CTFs

Si steghide pide passphrase, probar en este orden:
1. Vacía (solo Enter)
2. Nombre de la máquina
3. Nombre de usuario encontrado
4. `admin`, `password`, `1234`
5. Palabras clave del tema de la máquina

---

## Encodings comunes en CTFs

Si encuentras texto que no entiendes, puede estar codificado:

| Apariencia | Encoding | Herramienta |
|------------|----------|-------------|
| Solo `+`, `>`, `<`, `.`, `,`, `[`, `]` | Brainfuck | dcode.fr |
| Letras y números mezclados, largo | Base64 | CyberChef |
| Solo letras y números, sin `0`, `O`, `I`, `l` | Base58 | CyberChef |
| Solo 0 y 1 | Binario | CyberChef |
| Puntos y rayas | Morse | dcode.fr |

**CyberChef:** `https://cyberchef.org` — detecta automáticamente muchos encodings con "Magic"

**dCode:** `https://www.dcode.fr` — especializado en cifrados y códigos esotéricos

---

## Ejemplo real — Máquina Empire: LupinOne

La imagen `arsene_lupin.jpg` no contenía datos ocultos via esteganografía. Sin embargo, la información estaba oculta en el **código fuente HTML** de la página web como comentario, codificada en **Base58** (clave SSH privada).

Herramientas probadas sin resultado: steghide (no soportaba PNG), zsteg (sin resultados), binwalk (solo datos normales de la imagen).
