# John the Ripper

## Descripción

John the Ripper (John) es una herramienta de **crackeo de hashes offline**. Trabaja sin conectarse a ningún servicio — le das un hash y prueba contraseñas de un diccionario hasta encontrar la que genera ese hash. Es la herramienta estándar para crackear hashes de contraseñas y claves SSH protegidas.

**Diferencia clave con Hydra:**
- **John** → crackea hashes **offline** sin tocar la red
- **Hydra** → ataca servicios **online** conectándose a la red

---

## Instalación

Viene preinstalado en Kali Linux:

```shell
john --version
```

---

## Flujo básico

```
1. Obtener el hash (de /etc/shadow, clave SSH, etc.)
2. Convertirlo al formato que John entiende (si es necesario)
3. Lanzar John con una wordlist
4. Ver el resultado
```

---

## Crackear hashes de /etc/shadow

```shell
# Guardar el hash en un archivo
echo 'root:$y$j9T$HASH...:18919:0:99999:7:::' > hash.txt

# Crackear — John detecta el formato automáticamente
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Si no detecta el formato, especificarlo
john hash.txt --format=crypt --wordlist=/usr/share/wordlists/rockyou.txt

# Ver resultados
john hash.txt --show
```

> **Importante:** Usar siempre **comillas simples** al hacer echo del hash para evitar que bash interprete los `$` como variables.

---

## Crackear claves SSH protegidas con passphrase ⭐

Primero hay que convertir la clave al formato que John entiende con **ssh2john**:

```shell
# Convertir la clave privada a hash crackeable
ssh2john id_rsa > hash_rsa.txt

# Crackear con wordlist
john hash_rsa.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Para claves CTF usar fasttrack (más pequeña pero específica)
john hash_rsa.txt --wordlist=/usr/share/wordlists/fasttrack.txt
```

---

## Otros conversores — formato2john

John tiene conversores para muchos tipos de archivos:

```shell
ssh2john id_rsa > hash.txt           # Claves SSH
zip2john archivo.zip > hash.txt      # ZIPs protegidos
rar2john archivo.rar > hash.txt      # RARs protegidos
pdf2john archivo.pdf > hash.txt      # PDFs protegidos
keepass2john database.kdbx > hash.txt # KeePass
```

---

## Wordlists más usadas

| Wordlist | Tamaño | Cuándo usarla |
|----------|--------|---------------|
| `/usr/share/wordlists/rockyou.txt` | 14M | Primera opción siempre |
| `/usr/share/wordlists/fasttrack.txt` | Pequeña | CTFs — contraseñas comunes y patrones |
| `/usr/share/wordlists/metasploit/unix_passwords.txt` | Media | Sistemas Unix |

---

## Formatos de hash más comunes

| Prefijo | Algoritmo | Comando |
|---------|-----------|---------|
| `$1$` | MD5 | `--format=md5crypt` |
| `$5$` | SHA-256 | `--format=sha256crypt` |
| `$6$` | SHA-512 | `--format=sha512crypt` |
| `$2y$` | bcrypt | `--format=bcrypt` |
| `$y$` | yescrypt | `--format=crypt` |
| Sin prefijo (32 chars) | MD5 raw | `--format=raw-md5` |
| Sin prefijo (40 chars) | SHA1 | `--format=raw-sha1` |

---

## Ver resultados y gestión

```shell
# Ver contraseñas crackeadas
john hash.txt --show

# Listar formatos disponibles
john --list=formats

# Continuar un crackeo interrumpido
john hash.txt --restore

# Usar múltiples cores
john hash.txt --wordlist=rockyou.txt --fork=4
```

---

## Notas importantes

- John guarda el progreso automáticamente en `~/.john/john.pot`
- Los hashes **yescrypt** (`$y$`) son extremadamente lentos — pueden tardar días con rockyou.txt
- Si John dice `No password hashes loaded` → problema con el formato, especifica con `--format=`
- Las comillas dobles en echo interpretan `$` como variables — usar siempre comillas simples

---

## Ejemplo real — Máquina Empire: LupinOne

Crackeo de passphrase de clave SSH privada:

```shell
ssh2john clave_privada > hash_rsa.txt
john hash_rsa.txt --wordlist=/usr/share/wordlists/fasttrack.txt
# Resultado: P@55w0rd!
```

El mensaje en `/~secret` indicaba explícitamente usar fasttrack — en CTFs las pistas sobre qué wordlist usar son muy valiosas.

## Ejemplo real — Máquina Metasploitable 2

Crackeo de hash yescrypt de root:

```shell
echo 'root:$y$j9T$M3BDdkxYOlVM6ECoqwUFs.$Wyz40CNLlZCFN6Xltv9AAZAJY5S3aDvLXp0tmJKlk6A:18919:0:99999:7:::' > hash_root.txt
john --format=crypt --wordlist=/usr/share/wordlists/rockyou.txt hash_root.txt
# Resultado: muy lento — hash yescrypt moderno
```
