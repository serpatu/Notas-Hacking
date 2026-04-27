# Exiftool

**Categoría:** Forense / Esteganografía 
**Función:** Lectura y escritura de metadatos en archivos (imágenes, PDF, docx...). 
**Etiquetas:** #forense #metadatos #stego 
## Uso Común 
Muchas veces las pistas o flags están ocultas en los comentarios, coordenadas GPS o autor de una foto.
### Comandos Clave
**-Ver todos los metadatos:**
```bash exiftool imagen.jpg```
## 🛠️ Uso Avanzado (Hacking & Forense)

### 1. Escritura y Modificación (Anti-Forense)
Podemos alterar los metadatos para ocultar pistas o engañar a los analistas.

- **Cambiar el autor:**
```bash
exiftool -Artist="Hacker" imagen.jpg
```

- **Cambiar todas las fechas:**
```bash
exiftool -AllDates+="1:0:0 0" imagen.jpg
```

- **Cambiar todas las fechas:**
```bash
  exiftool -all= imagen.jpg
  ```