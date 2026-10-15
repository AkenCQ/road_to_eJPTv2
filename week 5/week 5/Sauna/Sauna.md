___
# Sauna Machine (Hack The Box)

IP atacante: 10.10.17.214
IP víctima: 10.129.95.180

Comprobamos conectividad con la máquina víctima

![[sauna_ping.png]]

TTL = 127 -> Máquina Windows

___
# RECONOCIMIENTO Y ENUMERACIÓN

Realizamos un escaneo con `nmap` para ver posibles puertos abiertos

![[sauna_ports.png]]

![[sauna_servicesv.png]]

Ya que vemos muchos puertosa abiertos, intuimos que estamos ante un Active Directory. Asi que iniciamos enumerando el protocolo SMB.
	*Herramientas para enumerar: nxc, smbmap, smbclient*

![[sauna_domain.png]]

Encontramos un dominio `EGOTISTICAL-BANK.LOCAL`, lo añadiremos al `/etc/hosts` con la IP de la máquina víctima. Ahora intentamos conectarnos con `rpcclient` ya que el puerto 135 está abierto, de tal forma que podemos enumerar usuarios y derivar a un ataque, pero tampoco podemos.

![[sauna_rpcclient.png]]

También observamos el puerto 389 que pertenece al servicio LDAP, asi que enumeramos.

```bash
ldapsearch -x -H ldap://10.129.95.180 -s base namingcontexts

# -x: Autenticación simple
# -H: Indicamos la URI
# -s base: Búsqueda en la entrada base (solo información del servidor, nada dentro del directorio)
# namingcontexts: Solicita el atributo `namingContexts`, que devuelve los dominios/ámbitos del directorio
```

![[sauna_ldapsearch.png]]

Buscamos todas las entradas del directorio LDAP bajo el dominio ``EGOTISTICAL-BANK.LOCAL``

```bash
ldapsearch -x -H ldap://10.129.95.180 -b 'DC=EGOTISTICAL-BANK,DC=LOCAL'

# -b <DN>: define el base DN desde dónde empezar la búsqueda
```

![[sauna_enumldap.png]]

Encontramos un ``CN=Hugo Smith``, asi que filtramos por los `CN`

![[sauna_ldapenum2.png]]

Y solo encontramos dicho usuario, por lo que intuimos que en el sistema podría ser `hugo.smith`, `hsmith`, `h.smith`, etc.

Ahora intentamos enumerar usuarios usando el puerto 88 (kerberos) para enviar solicitudes usando `kerbrute`. Para ello necesitamos un archivo con todas las combinaciones posibles.

![[sauna_users.png]]

![[sauna_kerbrute.png]]

Encontramos un usuario llamado `hsmith`, con esto sabemos cómo es la estructura de los usuarios: Primero la inicial y luego el apellido.

Entrando a la página web, podemos encontrar nombres de empleados que pertenecen a dicha compañía, pero como sabemos el patrón, podemos agregarlos al archivo creado previamente para aplicar `impacket-GetNPUsers` y aplicar un ataque `ASP Roasting`.

![[sauna_webusers.png]]

![[sauna_userlist.png]]

Es vulnerable ya que no tiene la autenticación previa de Kerberos

![[sauna_fsmith.png]]

Obtenemos dicho hash, lo copiamos en un archivo hash y luego lo mandamos a nuestra máquina para usar hashcat y descifrarlo.

![[sauna_hashcat.png]]

User: `hsmith`
Pass: `Thestrokes23`

Volvemos a enumear el protocolo `smb` con esas credenciales

![[sauna_smbclientcred.png]]

Ya que no podemos conectarnos por smb, el puerto 5985 estaba abierto, por lo que usamos `nxc` y si nos pone un ``pwn3d``, es porque el usuario forma parte de ``Remote Management Users``

![[sauna_winrmpwned.png]]

Obtenemos la flag del usuario dentro del escritorio del usuario FSmith

![[sauna_flaguser.txt.png]]

Ahora usamos una consola interactiva haciendo uso de la herramienta `evil-winrm`

![[sauna_evilwinrm.png]]

Si listamos usuarios del grupo `Remote Managemente Users`, podemos ver otro usuario existente o también podemos verlo con `net user`.

![[sauna_otheruser.png]]

Ahora nos descargamos el archivo ``winPEASx64.exe`` para ver posibles formas de escalar privilegios. Esto lo logramos mediante un servidor python y en Windows ejecutar `cerutil.exe`.

Ejecutamos la herramienta y nos reporta la credencial del usuario `svc_loanmgr:Moneymakestheworldgoround!`
![[sauna_winPEAS.png]]

Con `nxc` comprobamos si es un usuario válido

![[sauna_validuser.png]]

Ahora nos conectamos mediante `evil-winrm`

![[sauna_user2evilwinrm.png]]

No encontramos nada interesante, entonces usamos la herramienta `BloodHound`

```bash
neo4j console &>/dev/null & disown
bloodhound-start &>/dev/null & disown
```

Nos traemos el recurso `SharpHound.ps1` a la máquina víctima vía servidor de python.

![[sauna_sharphound.png]]

Ahora generamos el comprimido con la información del sistema

![[sauna_createzip.png]]

Nos mandamos el comprimido a nuestra máquina para subirlo a BloodHound

```bash
download C:\Windows\Temp\privesc\20261014214556_BloodHound.zip bloodhound.zip
```

Dada la información que nos da ``BloodHound``, podemos usar `impacket-secretsdump` que permite extraer credenciales y secretos sin necesidad de ejecutar ningún agente en el objetivo, de tal forma que obtenemos el hash del usuario Administrator.

![[sauna_hashadmin.png]]

Hash: 823452073d75b9d1cf70ebdf86c7f98e

Finalmente, usamos la herramienta impacket-psexec indicamos el usuario, la IP y le pasamos el hash, efectuamos un pass de hash y obtenemos la flag del root.

![[sauna_flagroot.png]]