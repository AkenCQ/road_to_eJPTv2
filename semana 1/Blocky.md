***

IP Host: 10.10.17.184
IP máquina: 10.129.66.14

# 1. Reconocimiento

Realizaremos reconocimiento activo para descubrir posibles puertos abiertos y a partir de dicha información, saber a qué nos estamos enfrentando.

Para ello utilizamos un nmap:

```bash
nmap -p- --open -sS -sV -T5 -n -Pn -vvv 10.129.66.14 -oG scanPorts

# -sS (Stealth Scan): No realiza el Three Way Handshake 
```

![[eJPTv2/imgs/blocky_nmap.png]]

Podemos ver los puertos **21,22,80 y 25565** abiertos y sus respectivas versiones. 

## FTP

Dado a que vemos el puerto ftp abierto, intentamos conectarnos a este con el usuario anonymous (que no pide contraseña).

![[eJPTv2/imgs/blocky_ftp.png]]

Pero no se puede dado a que necesitamos un usuario y contraseña.

## HTTP

Haremos un escaneo para ver el servicio web que está utlizando y así poder recolectar más información sobre el sistema.

![[eJPTv2/imgs/p80_blocky.png|700]]

Utiliza un **Apache httpd 2.4.18**, para saber el codename buscamos el launchpad  y nos pondrá que es **Ubuntu Xenial**

Ahora entramos a la página web y exploramos un poco por ella, pero antes debemos incluir el nombre del host para que nos aparezca la página

![[eJPTv2/imgs/host_blocky-1.png|700]]


Es una página de un servidor de minecraft y explorando encontramos que utiliza wordpress, tiene un panel de login, el archivo xmlrpc.php solo recibe peticiones POST (este archvio debe estar deshabilitado) y que la versión de Wordpress es 4.8 (según nos descargamos un archivo que encontramos en la web del servidor de Minecraft).

Podemos obtener más información con whatweb

```bash
whatweb http://blocky.htb
```

![[eJPTv2/imgs/blocky_whatweb.png]]

![[eJPTv2/imgs/blocky_information.png]]

En la página vemos al usuario "Notch" quien realizó una publicación, por lo que asumimos que es un usuario válido para Wordpress

![[eJPTv2/imgs/blocky_user.png]]

Y acertamos porque nos dice que la password para el usuario notch es incorrecta.
![[eJPTv2/imgs/blocky_notch.png|400x600]]

## SSH

Dado a que tenemos al usuario notch, podemos identificar también si es un usuario a nivel del sistema buscando un exploit para enumerar usuarios vía ssh

```bash
searchsploit ssh user enumeration
```

![[eJPTv2/imgs/blocky_searchsploit.png]]

Y encontramos exploit que coinciden con la versión que tenemos del servicio nmap del servidor (<>). Nos traemos el script a nuestra carpeta actual

```bash
searchsploit -m 45939
```

Este se ejecuta con python2, asi que lo usamos de la siguiente forma:

```bash
python2 45939.py 10.129.66.14 admin 2>/dev/null
# Nos pondrá que admin es un usuario inválido
python2 45939.py 10.129.66.14 root 2>/dev/null
# Nos pondrá que es un usuario válido
python2 45939.py 10.129.66.14 notch 2>/dev/null
# Nos pondrá que es un usuario válido
```

# 2. Ataque

Comenzamos haciendo un fuzzing a blocky.htb utilizando ffuf

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt:FUZZ -u "http://blocky.htb/FUZZ"
```

![[eJPTv2/imgs/blocky_fuzzing.png]]

Encontramos una ruta /plugin el cual tiene dos archivos, los descargamos.


![[eJPTv2/imgs/blocky_files.png|700]]

Los descargamos y los traemos al directorio de trabajo actual.

Utilizamos el siguiente programa para ver la estructura del archivo

```bash
java -jar /usr/share/jd-gui/jd-gui.jar
```

![[eJPTv2/imgs/blocky_jfuid.png|700]]

Y buscando en el archivo BlockyCore.class obtenemos el usuario y la contraseña

![[eJPTv2/imgs/blocky_class.png|800]]

Credenciales: ``root/8YsqfCTnvxAUeduzjNSXe22``

Probamos con root y no funciona, pero también tenemos el usuario `notch`

```bash
ssh notch@10.129.66.14
```

y logramos ingresar a la máquina y ver la flag del usuario

![[eJPTv2/imgs/blocky_notchssh.png]]

Buscamos formas de escalar privilegios con la información que nos devuelven los siguientes comandos:

```bash
find / -perm -4000 2>/dev/null # Buscar archivos con permisos SUID
getcap -r / 2>/dev/null #Buscar capabilities del sistema
id # Ver a qué grupos pertenece el usuario "notch"
```

Con el último nos muestra que pertenece al grupo de sudoers, por lo que simplemente podemos hacer un sudo su para cambiar a root y obtener la flag.

![[eJPTv2/imgs/blocky_sudo 1.png]]

![[eJPTv2/imgs/blocky_sudo.png]]