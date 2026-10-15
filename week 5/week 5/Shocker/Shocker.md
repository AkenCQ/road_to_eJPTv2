___
# Shocker Machine (Hack The Box)

IP atacante: 10.10.17.214
IP víctima: 10.129.71.21

Comprobamos si tenemos conectividad con la máquina

![[shocker_ping.png]]

TTL = 63 -> Máquina Linux
___
# RECONOCIMIENTO Y ENUMERACIÓN

Realizamos un escaneo con nmap para ver posibles puertos abiertos

![[shocker_nmap.png]]

[+] Puertos abiertos
	-> 80 | http | Apache httpd 2.4.18
	-> 2222 | ssh | OpenSSH 7.2p2

En la página web encontramos lo siguiente

![[shocker_web.png]]

Dado a que no encontramos nada más relevante, hacemos fuzzing para ver directorios

![[shocker_fuzzing.png]]

Encontramos un directorio ``cgi-bin``, en este se suele almacenar scripts dinámicos de la web

Si hacemos un fuzzing a dicho directorio, podemos encontrar un archivo `user.sh`, que cada vez que recargamos la página, se actualiza.

![[shocker_usersh.png]]

Interceptamos el tráfico con `tshark` para guardarlo en un archivo al ejecutar un script de `nmap`.

```bash
tshark -i tun0 -w captura.cap

# Ejecutamos el script
nmap --script http-shellshock --scripts-args uri=/cgi-bin/user.sh -p80 10.129.72.13
```

![[shocker_tshark.png]]

Ahora inspeccionamos el archivo aplicando un filtro para ver solo el contenido `http`, representamos en JSON y filtramos por `tcp.payload` que es donde esta la información importante. Convertimos esa cadena hexadecimal a texto con `xxd`.

![[shocker_shellshock.png]]

# EXPLOTACIÓN 

En las cabeceras podemos ver que se acontence el ataque, por ejemplo en el `User-Agent`, asi que podemos aprovecharnos de eso y ejecutar comandos.

![[shocker_attack.png]]

Para comprobar, usamos `whoami` para ver si nos devuelve el usuario del sistema

![[shocker_user.png]]

Ya que es exitoso, nos mandamos una reverse shell por el puerto 443 (es escucha con netcat)

![[shocker_shell.png]]

Obtenemos la flag del usuario

![[shocker_flaguser.png]]

# ESCALADA DE PRIVILEGIOS

Buscamos qué comandos podemos ejecutar con sudo sin necesidad de contraseña, encontrando ``perl`` y spawneando una bash con privilegios al usar sudo.

![[shocker_rootflag.png]]