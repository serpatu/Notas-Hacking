# Metodología General de Pentesting

## Descripción

Este documento es el flujo completo de un ataque de pentesting. Úsalo como mapa general y deriva a las metodologías específicas en cada fase.

---

## Fases del ataque

```
1. Reconocimiento
       ↓
2. Enumeración
       ↓
3. Explotación (acceso inicial)
       ↓
4. Post-Explotación
       ↓
5. Escalada de Privilegios
       ↓
6. Persistencia / Pillaje
```

---

## Fase 1 — Reconocimiento

El objetivo es descubrir qué está expuesto en la máquina objetivo.

### Escaneo de puertos

```shell
# Escaneo rápido de todos los puertos
nmap -p- --min-rate 5000 TARGET_IP

# Escaneo detallado de puertos encontrados con versiones y scripts
nmap -p 22,80,443 -sV -sC TARGET_IP

# Escaneo con detección de OS
nmap -p- -sV -sC -O TARGET_IP
```

### Qué buscar en el resultado de nmap

| Puerto | Servicio | Siguiente paso |
|--------|----------|---------------|
| 21 | FTP | Probar acceso anónimo, versión vulnerable |
| 22 | SSH | Fuerza bruta si no hay otro vector |
| 80/443 | HTTP/HTTPS | → Ver Enumeracion-Web.md |
| 139/445 | SMB | Enumerar shares, EternalBlue |
| 3306 | MySQL | Probar credenciales por defecto |
| 5985 | WinRM | → Evil-WinRM si tienes credenciales |

---

## Fase 2 — Enumeración

Según los puertos encontrados, profundiza en cada servicio. Ver metodologías específicas:

- Puerto 80/443 → **Enumeracion-Web.md**
- Shell obtenida → **Post-Explotacion.md**

---

## Fase 3 — Explotación

### Flujo de decisión

```
Identificar servicio y versión
          ↓
searchsploit "nombre versión"
          ↓
¿Hay exploit público?
   SÍ → Leer exploit → Adaptar → Lanzar
   NO → Buscar credenciales por defecto → Fuerza bruta → Vulnerabilidades de configuración
```

### Credenciales por defecto más comunes

| Servicio | Usuario | Contraseña |
|----------|---------|------------|
| FTP | anonymous | (vacía) |
| MySQL | root | (vacía) / root |
| Tomcat | admin/tomcat | admin/tomcat/s3cret |
| TikiWiki | admin | admin |
| phpMyAdmin | root | (vacía) |

### Recursos para buscar exploits

- `searchsploit nombre_servicio versión` — búsqueda local en Exploit-DB
- `https://www.exploit-db.com` — búsqueda online
- `https://book.hacktricks.xyz` — técnicas y payloads por servicio
- `https://nvd.nist.gov` — base de datos de CVEs

---

## Fase 4 — Post-Explotación

Una vez dentro → **Post-Explotacion.md**

---

## Fase 5 — Escalada de Privilegios

Según el sistema operativo:

- Linux → **Escalada-Privilegios-Linux.md**
- Windows → **Escalada-Privilegios-Windows.md**

---

## Fase 6 — Pillaje

Una vez conseguido root o SYSTEM, recopilar información valiosa:

```shell
# Linux
cat /etc/shadow                        # Hashes de contraseñas
cat /root/.bash_history                # Historial de comandos de root
find / -name "*.txt" 2>/dev/null       # Archivos de texto
find / -name "id_rsa" 2>/dev/null      # Claves SSH privadas
cat /etc/hosts                         # Otros hosts en la red

# Windows
type C:\Users\Administrator\Desktop\root.txt
dir C:\Users\ /s /b                    # Listar todos los usuarios
```

---

## Reglas mentales del pentesting

1. **No te enamores de un vector** — si llevas 30 minutos sin avanzar, cambia de vector
2. **Enumera antes de explotar** — cuanta más información tengas, mejores decisiones tomarás
3. **Anota todo** — IPs, puertos, versiones, credenciales encontradas, rutas interesantes
4. **Lee el exploit antes de lanzarlo** — entiende qué hace y qué efecto tiene en el sistema
5. **Verifica la conectividad** antes de culpar al exploit — comprueba que LHOST es correcto y el puerto está libre
