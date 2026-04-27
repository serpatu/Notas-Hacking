# Gobuster
**Categoría:** Enumeración Web / Fuzzing
**Función:** Herramienta de fuerza bruta para descubrir directorios, archivos (URIs) y subdominios DNS.
**Etiquetas:** #herramienta #web #fuzzing

---

## Modos Principales
Gobuster funciona mediante "modos" que se especifican al principio del comando:

1. **`dir`**: Modo Directorio (El más común). Busca carpetas y archivos ocultos en una web.
2. **`dns`**: Modo DNS. Busca subdominios (ej: `dev.empresa.com`).
3. **`vhost`**: Modo Virtual Host. Busca subdominios en el mismo servidor web (Virtual Hosting).

## Comandos Útiles

### Búsqueda de Directorios (Modo `dir`)

> [!example] Comando Básico
> ```bash
> gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt
> ```

**Flags importantes:**
- `-u`: URL objetivo.
- `-w`: Wordlist (lista de palabras a probar).
- `-x`: Extensiones a buscar (ej: `-x php,html,txt`).
- `-t`: Hilos (velocidad, por defecto 10).