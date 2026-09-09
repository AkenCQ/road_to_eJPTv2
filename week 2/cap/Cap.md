___
# Máquina CAP

IP máquina atacante : 10.10.17.184
IP máquina víctima: 10.129.67.2

Comprobamos que tengamos conexión con la máquina haciendo ping.

![[cap_ping.png]]

ttl = 63 -> Máquina Linux

___
# 1. Reconocimiento

Iniciamos con haciendo nmap hacia la IP víctima para realizar un escaneo de puertos posiblemente abiertos

```bash
sudo nmap -p- --open -T5 -sV -vvv -sS -n -Pn 10.129.67.2 -oG scanPorts
```

![[cap_nmap.png]]

Podemos observar que tiene 3 puertos abiertos:

- 21 FTP -> vsftpd 3.0.3
- 22 SSH -> OpenSSH 8.2p1
- 80 HTTP -> Gunicorn

Echamos un vistazo a la página web para ver qué contiene

![[cap_web.png|900]]


Es un Dashboard para monitorizar las conexiones del servidor.

En el panel de la izquierda hay un apartado llamado "Security Snapshot" el cual nos muestra una captura .pcap, la cual guarda una grabación exacta del tráfico de datos que circula por la red. En este caso mostrará 0 porque no hemos hecho nada en la red.

![[cap_pcap.png]]

En la URL nos muestra que los datos son de la sesión ``/data/1``, si volvemos a pinchar en el Security Snapshot, mostrará la nueva captura como ``/data/2``.

![[cap_data2.png]]

Si buscamos por la captura 0, nos mostrará que tiene contenido, por lo que descargamos dicho archivo para analizarlo y descubrir que podemos obtener. Para esta acción, utilizaremos tshark (también se puede usar WireShark).

```bash
tshark -r 0.pcap tcp.payload 2>/dev/null
```

![[cap_0pcap.png]]

Y podemos ver el tráfico de red en el cual se almacenaron las credenciales al momento de que el usuario ``nathan`` se autenticó en el servicio 
`ftp`. Podriamos ingresar al ftp, pero también tenemos un servicio ssh activo, por lo que podemos autenticarnos allí también.

![[cap_ssh.png]]

Ingrasamos exitosamente como el usuario `nathan`, entonces vemos la flag del usuario.

![[cap_usertxt.png]]

# 2. Escalar privilegios

Para escalar privilegios, seguimos el proceso de buscar archivos con permisos SUID, capabilities del sistema, subdominios y otras conexiones.

```bash
find / -perm -4000 -user root 2>/Dev/bull
getcap -r / 2>/dev/null
```

![[cap_getcao.png]]

Viendo las capabilities, hay una que permite establecer el uid de un usuario en python. Por lo scripteamos un archivo para cambiar le UID de nuestro usuario a 0 (el de root) y lanzar una bash.

![[cap_scriptpy.png]]

Damos permisos de ejecución con `chmod` y ejecutamos:

```bash
chmod +x test.py

python3 test.py
```

![[cap_final.png]]
