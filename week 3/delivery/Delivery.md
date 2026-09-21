___

# Máquina Devel (HTB)

**IP atacante: 10.10.17.214**
**IP víctima: 10.129.57.15**

Comprobamos conexión con la máquina:

![[delivery_ping.png]]

TLL -> 63 -> Linux

___
# 1. Reconocimiento y enumeración

Realizamos un escaneo de puertos con nmap para ver posibles abiertos.

![[delivery_scanports.png]]

Puertos abiertos:

- Puerto 22 | SSH | OpenSSH 7.9p1 Debian 10+deb10u2
- Puerto 80 | HTTP | nginx 1.14.2
- Puerto 8065 | Golang net/http server

Miramos la página web para examinarla y podemos ver que tenemos un `Contact Us` que nos muestra dos enlaces 

![[delivery_page80.png|900]]


![[delivery_contactus.png]]

Podemos ver enlaces a `HelpDesk` y a `MatterMost server`, una es para un sistema de tickets y la otra para un chat de empresas.

En `HelpDesk` creamos un ticket con cualquier asunto, esto nos dará un correo el cual registraremos en Mattermost y miramos en `View Ticket Thread` con las credenciales que hemos registrado al crear el ticket para aprovecharnos de esto y saltarnos la verificación del correo. Nos dará un enlace de verificación para el correo de Mattermost, al cual nos dirigimos e iniciamos sesión con la cuenta creada allí.

![[delivery_ticket.png|900]]

Al entrar a Mattermost veremos un chat privado en el cual se muestran credenciales y una nota sobre otra credencial

![[delivery_chatprivate.png|900]]

Credenciales: `maildeliverer:Youve_G0t_Mail!`

Ya que tenemos el puerto ssh abierto, nos conectamos con esas credenciales, obteniendo acceso a una consola.

![[delivery_ssh.png]]

También descubrimos la flag del user

![[delivery_userflag.png]]


Listamos procesos como `mattemost` y miramos la ruta donde está almacenado para ver su configuración.

![[delivery_psfaux.png]]

Y viendo el archivo `config.json` podemos extraer información, en este caso encontramos unas credenciales para un usuario en `mysql`, de tal forma que podemos listar la base de datos existentes.
(Podemos saber si esta instalado un motor de base de datos enumerando los usuarios en el archivo `/etc/passwd`).

![[delivery_mysql.png]]

Nos conectamos a la base de datos con el siguiente comando:

```bash
mysql -u mmuser -p
# Pedirá la contraseña
```

Listamos las bases de datos existentes y su contenido

```mysql
show databases;
user mattermost;
show tables;
describe Users;
select username,password from Users;
```

![[delivery_mysqlusers.png|700]]

Obtenemos los usuarios y los hashes, el que más llama la atención es el del usuario root, por lo que usaremos hashcat para desencriptar su contraseña (esta en `bycrpt`), pero recordemos que en los mensajes del chat decían que esta no se encuentra en el diccionario `rockyou.txt`, pero que jugando con reglas se podría descifrar. Guardamos el hash en un archivo `hash`.

Para ello, nos clonamos este repositorio de GitHub:

```bash
git clone https://github.com/stealthsploit/Optimised-hashcat-Rule.git
```

Entramos al directorio y creamos el diccionario

```bash
hashcat -r OneRuleToRuleThemAll.rule --stdout > wordlist.txt
```

Con el diccionario ya creado, vamos a crackear la contraseña con JohnTheRipper

```bash
john --wordlist=wordlists.txt hash
```

El resultados es: `PleaseSubscribe!21`

![[delivery_rootflag.png]]