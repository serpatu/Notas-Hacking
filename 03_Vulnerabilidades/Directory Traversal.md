# Directory Traversal (Path Traversal)

**Concepto:**
Vulnerabilidad que permite a un atacante acceder a archivos y directorios que están fuera de la carpeta raíz de la web.

**Mecanismo:**
Se utiliza la secuencia de caracteres `../` (punto-punto-barra) para "escalar" o retroceder directorios hasta llegar a la raíz del sistema.

**¿Cuántos `../` poner?**
Como a menudo desconocemos la profundidad de la estructura de carpetas del servidor, la técnica estándar es usar una cantidad excesiva (ej: 6-10 veces).
* **En Windows/Linux:** Si intentas retroceder más allá de la raíz, el sistema simplemente te mantiene en la raíz. Por tanto, `../../../../../../windows/win.ini` funciona igual aunque la web esté solo a 2 carpetas de profundidad.

**Archivos típicos de prueba (Proof of Concept):**
* **Windows:** `../../windows/win.ini` o `../../windows/system32/drivers/etc/hosts`
* **Linux:** `../../etc/passwd`