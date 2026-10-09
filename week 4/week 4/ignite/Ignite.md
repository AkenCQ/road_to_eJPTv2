___
# Ignite Machine | THM

IP atacante: 192.168.192.200
IP víctima: 10.129.156.195

Comprobamos conectividad con la máquina

![[ignite_ping.png]]

TTL = 62 -> Máquina Linux

# Reconocimiento y enumeración

Realizamos un nmap para escanear y ver posibles puertos abiertos.

![[ignite_nmap.png]]

![[ignite_servicesv.png]]

[+] Puertos abiertos
	-> 80 | http | Apache httpd 2.4.8.18 ((Ubuntu))

Exploramos la página web alojada en el puerto 80

![[ignite_80.png]]

![[ignite_credentials.png]]

Encontramos credenciales por defecto para ingresar al panel de administrador: `admin:admin`

El ser un CMS, procedemos a buscar vulnerabilidades para dicha versión con `searchsploit`

![[ignite_searchsploit.png]]

Usamos la tercera opción, asi que nos copiamos el archivo a nuestro directorio actual de trabajo.

Ejecutamos el exploit y nos mandamos una shell a nuestro equipo por el puerto 443 y aplicamos tratamiento de tty.

![[ignite_exploit.png]]

![[ignite_shell.png]]

![[ignite_nc.png]]

Obtenemos la flag del usuario.

![[ignite_flaguser.png]]

Enumeramos usuarios y vemos que hay uno mysql

![[ignite_mysqluser.png]]

Dentro de `/var/www/html` existen archivos de configuración, pero revisamos el de la base de datos `database.php`

![[ignite_databasephp.png]]

En este encontramos las credenciales de root, asi que probamos a usarlas para loguearnos como root en el máquina con `su`.
Ya logueados, obtenemos la flag de `root`.

![[ignite_root.png]]
