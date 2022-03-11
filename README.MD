# **Práctica Apache**
***
## **-Configuración inicial**  
Lanzamos el docker compose. Una vez arrancados los contenedores configuramos el servidor DNS siguiendo los pasos realizados en anteriores practicas.

El archivo de zona para utilizar durante esta práctica queda de la siguiente manera:

```
;
; BIND data file for local loopback interface
;
$TTL	604800
@	IN	SOA	prueba.com. root.prueba.com. (
			      2		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	      IN   NS	ns.prueba.com.
@         IN   A   10.0.0.88
@	      IN  AAAA ::1
ns        IN   A   10.0.0.88
maquina1  IN   A   10.0.0.74
www       IN CNAME maquina1.prueba.com.
```

Una vez configurado el DNS comprobamos que todo funciona correctamente. No es necesario realizar todavia ninguna configuracion en el servidor apache, simplemente abrimos un navegador de internet en nuestro host y entramos en https://localhost:6901, (necesario acceder a traves de https, el puerto es el que hayamos redireccionado durante la configuracion del Docker Compose) para acceder a la interfaz grafica del contenedor que actuará como cliente (en este caso un Kasm_Desktop). Introducimos el usuario y la contraseña (por defecto, si no hemos modificado la configuracion a traves de las variables de entorno en el docker compose, son kasm_user y password, respectivamente) y, una vez cargado el escritorio, abrimos un navegador de internet.

Utilizando el navegador accedemos a la IP del contenedor con el servidor web (10.0.0.74) y a las direcciones url que le hemos configurado en el servidor DNS (maquina1.prueba.com y su cname www.prueba.com). Si el servidor nos devuelve el mensaje de "it works" es que nuestro entorno funciona correctamente y podemos comenzar la configuracion del servidor apache.


## **-Configuración Apache y VirtualHost**

Abrimos el volumen "htdocs". Este volumen corresponde a "/usr/local/apache2/htdocs" que es el directorio que utiliza Apache para proporcionar el contenido web. Creamos dos carpetas y en su interior creamos un archivo index.html con el contenido html que queramos.

Abrimos el volumen "apacheconf", que corresponde a "/usr/local/apache2/conf" para acceder a las archivos de configuracion de apache. Entramos en la carpeta extra y abrimos el archivo httpd-vhosts.conf. Este archivo nos permite configurar los VirtualHost. Con ellos podemos hacer que nuestro Apache sirva distintas paginas web en funcion de la url que se solicite y utilizando una misma direccion IP. Estos son dos ejemplos de VirtualHost:

```
<VirtualHost *:80>
    ServerAdmin webmaster@dummy-host.example.com
    DocumentRoot "/usr/local/apache2/htdocs/prueba1"
    ServerName maquina1.prueba.com
    ErrorLog "logs/dummy-host.example.com-error_log"
    CustomLog "logs/dummy-host.example.com-access_log" common
</VirtualHost>

<VirtualHost *:80>
    ServerAdmin webmaster@dummy-host2.example.com
    DocumentRoot "/usr/local/apache2/htdocs/prueba2"
    ServerName www.prueba.com
    ErrorLog "logs/dummy-host2.example.com-error_log"
    CustomLog "logs/dummy-host2.example.com-access_log" common
</VirtualHost>
```

Basicamente las opciones a configurar son: 

**-ServerAdmin**: direccion de correo del administrador
**-DocumentRoot**: ruta en la que se encuentra el archivo index.html de cada virtual host
**-ServerName**: nombre de dominio del virtual host
**-ErrorLog**: ruta donde se almacenará el log de errores
**-CustomLog**: ruta donde se almacenará el log de acceso

Guardamos y abrimos el archivo httpd.conf, en el cual descomentamos la linea 524

```
Include conf/extra/httpd-vhosts.conf
```

Para incluir el archivo vhosts que acabamos de modificar dentro del archivo de configuración principal de Apache. Reiniciamos el servicio al finalizar.

De esta forma, al escribir la dirección url (maquina1.prueba.com o www.prueba.com) el servidor DNS nos enviará la direccion IP del servidor Apache y Apache atenderá cualquier peticion por el puerto 80 para servir el index.html correspondiente a la dirección que se le solicite en la peticion entrante.


## **-Configuración HTTPS**

HTTPS es un protocolo para transmitir hipertexto de forma segura. Se combina el protocolo HTTP con el protocolo SSL/TLS para realizar una conexión segura con el servidor y el cliente, asegurando que la información no pueda ser interceptada por terceras personas. 

Para habilitar HTTPS necesitamos una combinación de un certificado público y una clave privada. La clave SSL se mantiene secreta en el servidor y se utiliza para cifrar contenido que se envía a los clientes. El certificado SSL se comparte de forma pública con cualquiera que solicite el contenido y uede utilizarse para descifrar el contenido firmado por la clave SSL asociada. De este modo nos aseguramos de que se encripta la información y que esta llega tiene un origen y un destinatario real (es decir, que nadie está suplantando a nuestro servidor y que somos quien aseguramos ser).

Habitualmente este certificado sería respaldado por un organismo de certificación, pero para el desarrollo de la práctica utilizaremos un certificado autofirmado. Como estamos empleando Docker nuestro servidor no dispone de las herramientas necesarias para realizar esta tarea, por lo que generaremos la clave y el certificado en nuestra maquina host y posteriormente los trasladaremos al contenedor.

Para crear la clave y el certificado utilizamos el siguiente comando:

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/server.key -out /etc/ssl/certs/server.crt
```


**-openssl:** es la herramienta de línea de comandos básica para crear y administrar certificados, claves, y otros archivos de OpenSSL.
**-req:** este subcomando especifica que deseamos usar la administración de la solicitud de firma de certificados (CSR) X.509. El “X.509” es un estándar de infraestructura de claves públicas al que se adecuan SSL y TLS para la administración de claves y certificados a través de él. Queremos crear un nuevo certificado X.509, por lo que usaremos este subcomando.
**--x509:** modifica aún más el subcomando anterior al indicar a la utilidad que deseamos crear un certificado autofirmado en lugar de generar una solicitud de firma de certificados, como normalmente sucede.
**--nodes:** indica a OpenSSL que omita la opción para proteger nuestro certificado con una frase de contraseña. Necesitamos que Apache pueda leer el archivo, sin intervención del usuario, cuando se inicie el servidor. Una frase de contraseña evitaría que esto suceda porque tendríamos que ingresarla tras cada reinicio.
**--days 365:** esta opción establece el tiempo durante el cual el certificado se considerará válido. En este caso, lo configuramos por un año.
**--newkey rsa:2048:** especifica que deseamos generar un nuevo certificado y una nueva clave al mismo tiempo. No creamos la clave que se requiere para firmar el certificado en un paso anterior, por lo que debemos crearla junto con el certificado. La parte rsa:2048 le indica que cree una clave RSA de 2048 bits de extensión.
**--keyout:** esta línea indica a OpenSSL dónde colocar el archivo de clave privada generado que estamos creando.
**--out:** indica a OpenSSL dónde colocar el certificado que creamos.

Como se mencionó anteriormente, estas opciones crearán un archivo de clave y un certificado. Se harán algunas preguntas sobre nuestro servidor con el fin de insertar la información de forma correcta en el certificado.

Tras esto los copiaremos a nuestro contenedor con los comandos:

```
docker cp server.key nombrecontenedor:/usr/local/apache2/conf/
docker cp server.crt nombrecontenedor:/usr/local/apache2/conf/
```

Ahora debemos modificar nuevamente los archivos de configuración de nuestro servidor Apache. Abrimos httpd.conf y descomentamos la linea 541

```
Include conf/extra/httpd-ssl.conf
```

Abrimos el archivo conf/extra/httpd-ssl.conf y nos aseguramos de tener un virtual host configurado de la siguiente manera:

```
<VirtualHost _default_:443>
DocumentRoot "/usr/local/apache2/htdocs/prueba1"
ServerName maquina1.prueba.com
ServerAdmin you@example.com
ErrorLog /proc/self/fd/2
TransferLog /proc/self/fd/1
SSLEngine on
SSLCertificateFile "/usr/local/apache2/conf/server.crt"
SSLCertificateKeyFile "/usr/local/apache2/conf/server.key"
```

Es una configuración similar a la realizada en el apartado anterior de la práctica, solo que en este caso el virtual host escuchará en el puerto 443 (puerto por defecto para HTTPS) y debemos asegurarnos de proporcionar la ruta del certificado y de la clave. Asi mismo, debemos descomentar en el archivo httpd.conf las lineas que habilitan los siguientes modulos: mod_log_config, mod_setenvif, mod_ssl, socache_shmcb_module, socache_dbm_module

Guardamos todos los cambios, reiniciamos el servicio, accedemos a Kasm_Desktop y en el navegador de internet accedemos a https://maquina1.prueba.com

Recibiremos un aviso confirmandonos que deseamos acceder a la pagina (esto es consecuencia de utilizar un certificado autofirmado) y al aceptar la veremos en pantalla. A traves de las opciones del navegador podemos acceder a la información que hayamos introducido en nuestro certificado.



