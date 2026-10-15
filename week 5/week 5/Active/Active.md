___
# Active Machine (Hack The Box)

IP atacante: 10.10.17.214
IP víctima: 10.129.72.197

Comprobamos conectividad con la máquina

![[active_ping.png]]

TTL = 127 -> Máquina Windows

___

# RECONOCIMIENTO Y ENUMERACIÓN

Realizamos un escaneo de puertos con `nmap` para ver cuáles están abiertos

![[active_nmap.png]]

![[active_servicesv.png]]

Al ver tantos puertos abiertos y tratarse de Windows, podemos intuir que estamos antes un `Domain Controller`, asi que procedemos a enumerar el servicio samba para obtener usuarios, carpetas compartidas, etc.

Primero usamos `smbclient` para ver dichos recursos compartidos, aplicando una sesión anónima.

![[active_dirshares.png]]

Podemos ver disitintos directorios, pero el que nos llama la atención es `Replication`, asi que entramos en él como anónimo para enumerar cosas.

![[active_enumReplication.png]]

Explorando, encontramos un archivo `Groups.xml`, asi que lo traemos a la máquina para ver qué contiene

![[active_enumReplication2.png]]

Dentro de ese archivo podemos ver un usuario y una contraseña que está encriptada.
![[active_grourpsxml.png]]

Usuario: SVC_TGS
Contraseña (encriptada): edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ

Hacemos uso de `gpp-decrypt` para desencriptar la contraseña. 
	*Esta herramienta se usa para descrifrar contraseñas almacenadas en archivos de Preferencias de Directiva de Grupo de Windows (GPP). Permite recuperar credenciales que los administradores de sistemas han configurado para automatizar tareas, las cuales están protegidas mediante una encriptación AES-256 con una clave pública conocida (publicada por Microsoft).*

![[active_gppdecrypt.png]]

Contraseña: GPPstillStandingStrong2k18

Ahora que tenemos credenciales validas, podemos usar `nxc` o directamente `smbclient` para volver a listar recursos compartidos, pero autenticados, lo que nos permitiría ver más a detalle ciertas carpetas y o obtener la flag del usuario.

![[active_flaguser.png]]

Ahora nos conectamos por `rpcclient` para listar usuarios y demás información con las credenciales obtenidas.
	*Tip: Hay casos en lo que podemos aplicar un Null Session y conectarnos directamente: rpcclient -U "" (IP) -N*

![[active_rpcclient.png]]

Enumeramos usuarios y nos quedamos con el `rid` del grupo `Domain Admins` porque es donde hay usuarios con privilegios.

![[active_groups.png]]

![[active_userprivilege.png]]

Y solo encontramos al usuario `Administrador`
	*Tip: También podemos ver una descripción, en caso de querer verla, usamos: querydispinfo*

Probamos un a realizar un `AS-REP Roasting attack`. Para ello necesitamos usuarios válidos en un archivo, usaremos `SVC_TGS`. Este ataque consisten en que si no disponemos de credenciales de los usuarios, especificamos un diccionario de estos y se efectuará una consulta, la cual nos proporcionará un hash que podemos descifrar por fuerza bruta. Para realizarlo, la hora de nuestro equipo tiene que estar sincronizada con la hora del AD, asi que hacemos un `ntpdate <IP>`

Ya que tenemos el puerto 88 (kerberos) abierto, también podemos usar `kerbrute` para enumerar usuarios válidos y derivar a un `AS-REP Roasting Attack`.

![[active_asreproast.png]]

No tiene la opción acitvada, asi que no se puede.

Ahora probamos un ataque `Kerberoasting`, utilizamos la herramienta `impacket-GetUserSPNs`. Este ataque consiste en identificar cuentas de usuario que tienen asignados `Service Principal Names (SPN)` y solicitar tickets de concesión de servicio (TGS) cifrados con la contraseña de dichas cuentas.

![[active_kerberoasting.png]]

Vemos el usuario Administrator, por lo que podemos obtener un ``Ticket Granding Servicer (TGS)``, asi que aplicamos un `-request`

![[active_hashAdministrator.png]]

Obtenemos el hash y lo guardamos en un archivo, ahora debemos descifrarlo para obtener la contraseña del usuario ``Administrator``.

Usando la herramienta `john` con el diccionario `rockyou.txt` y especificando el tipo de hash, descifra la contraseña.

![[active_passAdministrator.png]]

User: `Administrator:Ticketmaster1968`

Y así, nos conectamos con dicho usuario usando `smbclient`, nos dirigimos a la carpeta específica y obtenemos la flag del root.

![[active_flagroot.png]]