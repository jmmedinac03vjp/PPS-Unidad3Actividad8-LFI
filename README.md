# PPS-Unidad3Actividad7-RCE
Explotación y Mitigación de ().
Tenemos como objetivo:

> - Ver cómo se pueden hacer ataques .
>
> - Analizar el código de la aplicación que permite ataques de .
>
> - Implementar diferentes modificaciones del codigo para aplicar mitigaciones o soluciones.

## ¿Qué es Inclusión de archivos locales (LFI)?
---

La vulnerabilidad de Inclusión de archivos permite a un atacante incluir un archivo, generalmente explotando un mecanismo “dynamic file inclusion” implementado en la aplicación de destino. La vulnerabilidad se produce debido al uso de la entrada suministrada por el usuario sin la validación adecuada.

Esto puede conducir a algo como la salida del contenido del archivo, pero dependiendo de la gravedad, también puede conducir a:

- Ejecución de código en el servidor web

- Ejecución de código en el lado del cliente, como JavaScript, que puede conducir a otros ataques, como secuencias de comandos en sitios cruzados (XSS)

- Denegación de Servicio (DoS)

- Divulgación de Información Sensible

La inclusión local de archivos (también conocida como LFI) es el proceso de incluir archivos, que ya están presentes localmente en el servidor, a través de la explotación de procedimientos de inclusión vulnerables implementados en la aplicación. Esta vulnerabilidad ocurre, por ejemplo, cuando una página recibe, como entrada, la ruta al archivo que debe incluirse y esta entrada no se desinfecta correctamente, lo que permite inyectar caracteres de salto de directorio (como punto-punto-slash). Aunque la mayoría de los ejemplos apuntan a scripts PHP vulnerables, debemos tener en cuenta que también es común en otras tecnologías como JSP, ASP y otras.
-
## ACTIVIDADES A REALIZAR
---

> Lee detenidamente la sección de Local File  de la página de PortWigger <https://portswigger.net/web-security/file-path-traversal>

> Lee el siguiente [documento sobre Explotación y Mitigación de ataques de Remote Code Execution](./files/ExplotacionYMitigacionLFI.pdf>
 
> También y como marco de referencia, tienes [ la sección de correspondiente de ataque LFI Local File Inclusion **Proyecto Web Security Testing Guide** (WSTG) del proyecto **OWASP**.]<https://owasp.org/www-project-web-security-testing-guide/stable/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion> 


En esta ocasión vamos a recrear los archivos que tenemos para el entrenamiento de esta vulnerabilidad en la [máquina DVWA](https://github.com/digininja/DVWA)

Vamos realizando operaciones:

## Código vulnerable
---
Tenemos el archivo vulnerable lfi.php con el siguiente contenido.
~~~
<?php

$page[ 'body' ] .= "
<div class=\"body_padded\">
	<h1>Vulnerability: File Inclusion</h1>

	{$WarningHtml}

	<div class=\"vulnerable_code_area\">
		[<em><a href=\"?page=file1.php\">file1.php</a></em>] - [<em><a href=\"?page=file2.php\">file2.php</a></em>]
	</div>
</div>\n";

?>

~~~
Este archivo nos muestra para hacer click en dos elementos que nos llevan a dos páginas diferentes: file1.php y file2.php con el siguiente contenido


file1.php
~~~
<?php

$page[ 'body' ] .= "
<div class=\"body_padded\">
	<h1>Vulnerability: File Inclusion</h1>
	<div class=\"vulnerable_code_area\">
		<h3>File 1</h3>
		<hr />
		Hello <em>" . tio . "</em><br />
		Your IP address is: <em>{$_SERVER[ 'REMOTE_ADDR' ]}</em><br /><br />
	</div>
</div>\n";

?>

~~~
Esta página nos muestra nuestra dirección ip.

El archivo file2.php tiene el siguiente contenido: 

~~~
<?php

$page[ 'body' ] .= "
<div class=\"body_padded\">
	<h1>Vulnerability: File Inclusion</h1>
	<div class=\"vulnerable_code_area\">
		<h3>File 2</h3>
		<hr />
		\"<em>I needed a password eight characters long so I picked Snow White and the Seven Dwarves.</em>\" ~ Nick Helm"<br /><br />

</div>;

?>

![](files/lfi1.png)

Al pulsar en los enlaces nos muestra el contenido de los archivos

![](files/lfi2.png)

##Explotación de LFI
---

**Leer archivos sensibles**
Prueba básica: Teniendo en cuenta que la página debe de estar en directorio /var/www/html podemos intentar ponerle la ruta de un archivo, tanto en ruta absoluta como relativa. Por ello si ponemos la ruta de /etc/passwd podemos intentar ver los datos de usuarios del sistema


~~~
http://localhost/lfi.php?file=../../../../etc/passwd
~~~
Si devuelve similar a:

![](files/lfi3.png)
 
La aplicación es vulnerable a LFI.

**Ejecutar código PHP (PHP Wrappers)**
---
Si la aplicación permite incluir archivos .php, se podría ejecutar código malicioso.

Ejemplo con php://filter para leer código fuente de archivos PHP: Vamos a intentar extraer el código del archivo index.html del servidor, aunque podríamos acceder a cualquiera:

Para que funcione este ataque tienen que estar activada la función de PHP allow_url_include. 

Si tenemos levantado el escenario en docker tenemos que acceder al contenedor del servidor web:

~~~
docker exec -it lamp-php83  /bin/bash
~~~

y editamos el archivo de configuración de php. Recordamos que en nuestro contendor está en una ruta diferente.

~~~
nano /usr/local/etc/php/php.ini
~~~

y ponermos la variable allow_url_include a on.

~~~
allow_url_include=on
~~~

Después reiniciamos el servicio.

![](files/lfi5.png)

Vamos a realizar el ataque, para ver si podemos ejecutar php en el servidor y así obtener el código del archivo index.html
~~~
http://localhost/lfi.php?file=php://filter/convert.base64-encode/resource=index.html
~~~


- file= → Se usa un parámetro vulnerable en lfi.php que permite incluir archivos arbitrarios en el servidor.php://filter/convert.base64-encode/resource=index.html:
- php://filter → Es un wrapper especial de PHP que permite aplicar filtros de manipulación de datos.
	- convert.base64-encode → Codifica el contenido del archivo en Base64 en lugar de mostrarlo directamente.
	- resource=index.html → Especifica el archivo que se quiere leer, en este caso index.html.

![](files/lfi4.png)

**Decodificar la cadena en Base64 y mostrar el resultado:**

Copiamos la cadena que hemos obtenido y realizamos la decodificación:

~~~
echo "BASE64_ENCODED_DATA" | base64 -d
~~~
![](files/lfi5.png)

Es posible que podamos encontrar en este u otro archivo el comentario con la contraseña, etc....

**Remote Code Execution (RCE) con Log Poisoning**
Si la aplicación escribe entradas de usuario en logs, se podría inyectar código PHP malicioso en ellos y luego incluir el archivo de logs.

Enviar payload en User-Agent (inyectar en logs de Apache). Ejecutamos en un terminal de comandos

~~~
curl -A "<?php system('whoami'); ?>" http://localhost
~~~

Hacemos un LFI para  Incluir el log para ejecutar código:

![](files/lfi7.png)

~~~
http://localhost/lfi.php?file=/var/log/apache2/access.log
~~~
o si lo tenemos en la pila LAMP de docker en:
~~~
http://localhost/lfi.php?file=/var/log/apache2/other_vhosts_access.log
~~~
 Si se muestra el resultado de **whoami**, LFI ha escalado a la ejecución de comandos (RCE).

### Mitigación de LFI
---

**Usar una Lista Blanca de Archivos Permitidos**
---
Una primera mitigación que podemos realizar es una lista blanca, de manera que sólo podamos incluir los archivos de una lista. En nuestro caso: file1.php y file2.php
~~~
<?php
// Establecemos una lista de archivos que se pueden incluir
$whitelist = ["home.php", "contact.php"];
if (isset($_GET['file'])) {
        $file = $_GET['file'];
        if (!in_array($file, $whitelist)) {
                die("Acceso denegado.");
        }
        echo file_get_contents($file);
}

?>

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Ejemplo de Enlaces</title>
</head>
<body>
    <h1>Elige un archivo</h1>
    <ul>
        <li><a href="?file=file1.php">Archivo 1</a></li>
        <li><a href="?file=file2.php">Archivo 2</a></li>
    </ul>
</body>
</html>
~~~

Si intentamos incluir cualquier otro archivo nos dá acceso denegado:
 
![](files/lfi8.png)

**Bloquear Secuencias de Directorios (../)**
---
En este caso la estrategia es tener un único o únicos directorios desde los que incluir los archivos. Sólo podrían estar en esa ubicación. En este caso le decimos que es en directorio donde se encuentra el archivo lfi.php
~~~
<?php
// Establecemos el directorio permitido en el mismo directorio del script
$baseDir = __DIR__ . DIRECTORY_SEPARATOR;

if (isset($_GET['file'])) {
    $file = $_GET['file'];

    // Normalizamos la ruta para evitar ataques con '../'
    $filePath = realpath($baseDir . $file);
    // Verificamos si el archivo está dentro del directorio permitido
    if ($filePath === false || strpos($filePath, $baseDir) !== 0) {
        die("Acceso denegado.");
    }

    // Verificamos que el archivo realmente existe
    if (!file_exists($filePath)) {
        die("El archivo no existe.");
    }
    echo file_get_contents($file);

}
?>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Ejemplo de Enlaces</title>
</head>
<body>
    <h1>Elige un archivo</h1>
    <ul>
        <li><a href="?file=file1.php">Archivo 1</a></li>
        <li><a href="?file=file2.php">Archivo 2</a></li>
    </ul>
</body>
</html>
~~~

**Restringir el Tipo de Archivo**
---

Si solo se permiten archivos .php, filtrar la extensión:

<?php
$file = $_GET['file'];
if (!preg_match('/^[a-zA-Z0-9_-]+\.php$/', $file)) {
die("Acceso denegado.");
}
include("pages/" . $file);
?>

Bloquea archivos con extensiones no deseadas.

**Deshabilitar allow_url_include y allow_url_fopen en php.ini**
---

allow_url_include = Off
allow_url_fopen = Off
Evita ataques de Remote File Inclusion (RFI).

**Usar realpath() para Evitar Path Traversal**

~~~
<?php
$file = $_GET['file'];
$baseDir = realpath("pages/");
$realPath = realpath("pages/" . $file);

if (strpos($realPath, $baseDir) !== 0) {
die("Acceso denegado.");
}
include($realPath);
?>
~~~

Garantiza que el archivo esté dentro de pages/.

![](files/lfi1.png)
![](files/lfi1.png)
![](files/lfi1.png)
![](files/lfi1.png)
![](files/lfi1.png)
![](files/lfi1.png)

~~~

## ENTREGA

>__Realiza las operaciones indicadas__

>__Crea un repositorio  con nombre PPS-Unidad3Actividad6-Tu-Nombre donde documentes la realización de ellos.__

> No te olvides de documentarlo convenientemente con explicaciones, capturas de pantalla, etc.

>__Sube a la plataforma, tanto el repositorio comprimido como la dirección https a tu repositorio de Github.__

