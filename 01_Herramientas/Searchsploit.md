# Searchsploit

## Descripción

Searchsploit es una **herramienta de línea de comandos para buscar exploits localmente** dentro de la base de datos de Exploit-DB. Permite encontrar exploits sin necesidad de conexión a internet, lo que es útil en entornos de laboratorio o redes aisladas.

---

## Instalación

Viene preinstalada en Kali Linux. Para actualizar la base de datos:

```shell
sudo searchsploit -u
```

---

## Uso básico

### Buscar por nombre de software

```shell
searchsploit tikiwiki
searchsploit twiki
searchsploit vsftpd
```

### Buscar por software y versión

```shell
searchsploit twiki 2003
searchsploit "linux kernel 2.6.24"
```

### Buscar por tipo de vulnerabilidad

```shell
searchsploit udev privilege escalation
searchsploit apache remote code execution
```

---

## Ver el contenido de un exploit

```shell
cat /usr/share/exploitdb/exploits/php/webapps/16892.rb
```

O con la opción `-x` (examinar) que muestra el contenido directamente:

```shell
searchsploit -x php/webapps/16892.rb
```

---

## Copiar un exploit a tu directorio de trabajo

```shell
searchsploit -m php/webapps/16892.rb
```

Esto copia el archivo al directorio actual, útil para modificarlo antes de usarlo.

---

## Filtrar resultados

### Solo exploits (excluir shellcodes)

```shell
searchsploit --exploit twiki
```

### Buscar por CVE

```shell
searchsploit cve-2009-1185
```

### Buscar por plataforma

```shell
searchsploit --type webapps tikiwiki
```

---

## Interpretar los resultados

La salida de searchsploit tiene dos columnas:

```
Exploit Title                                    | Path
-------------------------------------------------|----------------------------------
TWiki History TWikiUsers - 'rev' Cmd Execution   | php/webapps/16892.rb
TWiki 20030201 - 'search.pm' Remote Cmd Exec     | cgi/webapps/642.pl
```

- **Exploit Title**: Nombre descriptivo con software, versión y tipo de vulnerabilidad
- **Path**: Ruta relativa dentro de `/usr/share/exploitdb/exploits/`

---

## Consejos importantes

- Fíjate siempre en la **versión** del exploit vs la versión del software objetivo.
- Los archivos `.rb` son módulos de Metasploit, los `.py` son scripts Python, los `.c` son exploits en C que hay que compilar, y los `.txt` son PoC manuales.
- Cuando encuentres un exploit `.c` de kernel antiguo, es probable que necesites **compilarlo en la máquina víctima** en lugar de en Kali, ya que los headers del kernel moderno son incompatibles.

---

## Ejemplo real — Máquina Metasploitable 2

Búsquedas realizadas durante la resolución:

```shell
searchsploit tikiwiki      # → exploit 2701.txt (SQLi en sort_mode)
searchsploit twiki         # → exploits 16892.rb, 16894.rb, 642.pl
searchsploit vmsplice      # → exploits 5092.c, 5093.c (kernel 2.6.24)
searchsploit udev 2.6      # → exploit 8572.c (CVE-2009-1185) ← el que funcionó
```
