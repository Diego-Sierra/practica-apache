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
    ServerName aquina1.prueba.com
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

**-ServerAdmin**: 

