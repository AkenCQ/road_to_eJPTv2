___
# Lame Machine | HTB

IP atacante: 10.10.17.214
IP víctima: 10.129.69.176

Comprobamos conectividad con la máquina:

![[lame_ping.png]]

TTL = 63 -> Máquina Linux

___
# RECONOCIMIENTO Y ENUMERACIÓN

Realizamos un escaneo de puertos para ver cuáles están abiertos

![[lame_nmap.png]]

![[lame_servicesv.png]]

[+] Puertos abiertos
	-> 21 | ftp | vsftpd 2.3.4
	-> 22 | ssh | OpenSSH 4.7p1
	-> 139 | netbios-ssn | Samba smbd 3.X - 4.X
	-> 445 | microsoft-ds | Samba smbd 3.0.20-Debian
	-> 3632 | distccd | distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))

Procedemos a enumerar el servicio `samba` para ver recursos compartidos en la red

![[lame_smbclient.png]]

Ya que vemos la versión de samba `smbd 3.0.20`, podemos buscar vulnerabilidades con `searchsploit`

![[lame_searchploit.png]]

Lo que nos intersa es lograr RCE, asi que tomamos el segundo script que está programado en Ruby para ver qué payload inyecta.

![[lame_scriptpayload.png]]

Podemos ver que inyecta `"/=nohup"` en un campo username, asi que podemos usar el comando `logon` para conectarnos mediante un recurso compartido.

# EXPLOTACIÓN

Nos conectamos al recurso compartido `/tmp` visto anteriormente y obtenemos una shell con el comando `logon`.

Nos ponemos en escucha por el puerto 443 con `netcat`

```bash
nc -nlvp 443
```

```bash
smbclient //10.129.70.127/tmp
```

![[lame_usepayload.png]]

![[lame_flags.png]]
