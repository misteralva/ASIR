# Despliegue de Servidor Web Nginx con PHP y MySQL integrado en Red Local (DHCP + DNS)

**Módulo Profesional:** 0375 – Servicios en red  
**Tema:** B4 – Servidores Web  
**Ciclo:** CFGS Administración de Sistemas Informáticos en Red (ASIX)  
**Alumno:** David  

---

## 1. Arquitectura y Topología de Red

Para esta práctica se ha diseñado una infraestructura de dos máquinas virtuales en VirtualBox conectadas mediante una **Red Interna** aislada (`10.10.10.0/24`) bajo el dominio de pruebas `empresa.test`:

| Máquina Virtual | Sistema Operativo | Interfaces de Red | IP Asignada | Servicios Activos |
| :--- | :--- | :--- | :--- | :--- |
| **Router / Infraestructura** | Ubuntu Server (con entorno gráfico) | `enp0s3` (NAT)<br>`enp0s8` (Red Interna) | `10.0.2.15/24` (Internet)<br>`10.10.10.1/24` (LAN) | `kea-dhcp4-server` (DHCP)<br>`bind9` (DNS)<br>`iptables` (NAT/Router)<br>Firefox (Cliente de pruebas) |
| **Servidor Web (`nginx-server`)** | Ubuntu Server 24.04 LTS | `enp0s3` (Red Interna) | `10.10.10.20/24` (Reserva DHCP) | `nginx` (Puerto 80)<br>`php8.5-fpm` (FastCGI)<br>`mysql-server` (Base de datos) |

### ¿Por qué esta configuración de red?
* **Aislamiento y realismo:** La máquina `nginx-server` tiene **un único adaptador en Red Interna**. No dispone de adaptador NAT propio; obtiene su dirección IP, puerta de enlace y servidor DNS automáticamente desde el servidor **Kea DHCP**, y sale a Internet a través del enmascaramiento (`MASQUERADE`) configurado en `iptables` del servidor Router.
* **Resolución de nombres centralizada:** Al contar con un servidor **bind9** propio en `10.10.10.1`, no es necesario modificar el fichero `/etc/hosts` de ningún equipo. Todos los subdominios (`web1`, `web2`, `php`, `bd`) se resuelven dinámicamente mediante DNS.
* **Uso del dominio `.test`:** Se eligió `empresa.test` en lugar de `.local` porque `.local` está reservado para el protocolo mDNS (*Multicast DNS*) en sistemas Linux y puede provocar conflictos con `systemd-resolved`.

---

## 2. Preparación de Infraestructura (DHCP, DNS y Acceso SSH)

### 2.1. Reserva DHCP en Kea (`/etc/kea/kea-dhcp4.conf`)
Se configuró el servidor Kea en la máquina Router (`10.10.10.1`) para entregar como servidor DNS la propia IP del router y asignar una IP estática (`10.10.10.20`) fuera del pool dinámico (`10.10.10.2` – `10.10.10.15`) vinculada a la dirección MAC del adaptador interno de `nginx-server`.

```json
"option-data": [
    { "name": "routers", "data": "10.10.10.1" },
    { "name": "domain-name-servers", "data": "10.10.10.1" },
    { "name": "domain-name", "data": "empresa.test" }
],
"reservations": [
    {
        "hw-address": "08:00:27:XX:XX:XX",
        "ip-address": "10.10.10.20",
        "hostname": "nginx"
    }
]
```

* **`routers`:** Indica a los clientes que `10.10.10.1` es su puerta de enlace para salir a Internet.
* **`domain-name-servers`:** Obliga a los clientes a consultar nuestro `bind9` en lugar de DNS públicos externos.
* **`reservations`:** Garantiza que el servidor web mantenga siempre la IP `10.10.10.20` para que los registros DNS nunca queden desactualizados.

> **[INSERTAR CAPTURA: Salida de `ip -br a` y `ping -c 2 web1.empresa.test` en `nginx-server`]**

---

### 2.2. Zona Directa e Inversa en bind9
En `/etc/bind/named.conf.local` se declararon las zonas directa e inversa para `empresa.test`:

```bind
zone "empresa.test" IN {
    type master;
    file "/etc/bind/zones/db.empresa.test";
};

zone "10.10.10.in-addr.arpa" IN {
    type master;
    file "/etc/bind/zones/db.10.10.10";
};
```

#### Fichero de zona directa (`/etc/bind/zones/db.empresa.test`)
```bind
$TTL    604800
@       IN      SOA     ns1.empresa.test. admin.empresa.test. (
                        2026111601      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
;
@       IN      NS      ns1.empresa.test.
ns1     IN      A       10.10.10.1
nginx   IN      A       10.10.10.20
web1    IN      A       10.10.10.20
web2    IN      A       10.10.10.20
php     IN      A       10.10.10.20
bd      IN      A       10.10.10.20
```

* **Registros `A`:** Todos los subdominios (`web1`, `web2`, `php`, `bd`) apuntan a la misma dirección IP (`10.10.10.20`). Nginx se encargará posteriormente de distinguir qué web servir leyendo la cabecera HTTP `Host`.

#### Fichero de zona inversa (`/etc/bind/zones/db.10.10.10`)
```bind
$TTL    604800
@       IN      SOA     ns1.empresa.test. admin.empresa.test. (
                        2026111601      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
;
@       IN      NS      ns1.empresa.test.
1       IN      PTR     ns1.empresa.test.
20      IN      PTR     nginx.empresa.test.
```

> **[INSERTAR CAPTURA: Ficheros `db.empresa.test`, `db.10.10.10` y comprobación con `dig`]**

---

### 2.3. Administración Remota mediante Salto SSH (ProxyJump)
Dado que `nginx-server` se encuentra en una red interna aislada del equipo anfitrión (Windows), se configuró una regla de reenvío de puertos en el adaptador NAT del Router (`127.0.0.1:2222` $\rightarrow$ `puerto 22`) y se utilizó el Router como host bastión mediante `ProxyJump`:

```bash
ssh -J david@127.0.0.1:2222 david@10.10.10.20
```

---

## 3. Parte Práctica: Instalación y Configuración de Nginx

### 3.1. Dos Webs Estáticas en el mismo Dominio Base y Puerto 80 (2 puntos)

Se instaló Nginx y se crearon los directorios raíz para ambos sitios web:

```bash
sudo apt update && sudo apt install nginx -y
sudo mkdir -p /var/www/web1.empresa.test/html /var/www/web2.empresa.test/html
sudo chown -R $USER:$USER /var/www
```

Se crearon dos archivos `index.html` diferenciados para `web1` y `web2`, y se configuraron sus respectivos bloques de servidor (*server blocks*) en `/etc/nginx/sites-available/`:

#### Configuración de `web1.empresa.test` (`/etc/nginx/sites-available/web1.empresa.test`)
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name web1.empresa.test;

    root /var/www/web1.empresa.test/html;
    index index.html;

    access_log /var/log/nginx/web1.access.log;
    error_log  /var/log/nginx/web1.error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### Explicación de las directivas utilizadas:
* **`listen 80;` y `listen [::]:80;`:** Indican a Nginx que escuche peticiones HTTP estándar en el puerto `80` tanto en IPv4 como en IPv6.
* **`server_name web1.empresa.test;`:** Es la directiva clave del *Virtual Hosting* basado en nombres. Permite alojar múltiples sitios web compartiendo la misma IP y el mismo puerto (`80`); Nginx inspecciona la cabecera `Host` enviada por el navegador y selecciona el bloque que coincide.
* **`root /var/www/...;`:** Define el directorio del sistema de archivos donde se alojan los documentos públicos de esa web.
* **`index index.html;`:** Establece el archivo que se servirá por defecto cuando el usuario solicite la raíz (`/`) o un subdirectorio.
* **`access_log` y `error_log`:** Separan los registros de tráfico y errores de cada sitio web en archivos independientes dentro de `/var/log/nginx/`, facilitando la depuración.
* **`try_files $uri $uri/ =404;`:** Comprueba en orden si existe el archivo exacto solicitado (`$uri`), si existe una carpeta con ese nombre (`$uri/`) o, en caso contrario, devuelve un código de estado HTTP `404 Not Found`.

Se replicó la configuración para `web2.empresa.test`, se crearon los enlaces simbólicos en `/etc/nginx/sites-enabled/` (desactivando `default`) y se validó la sintaxis:

```bash
sudo sed 's/web1/web2/g' /etc/nginx/sites-available/web1.empresa.test | sudo tee /etc/nginx/sites-available/web2.empresa.test > /dev/null
sudo ln -s /etc/nginx/sites-available/web1.empresa.test /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/web2.empresa.test /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

> **[INSERTAR CAPTURA: Ficheros `web1.empresa.test` y `web2.empresa.test` en terminal]**  
> **[INSERTAR CAPTURA: Ambas webs abiertas en Firefox mostrando URLs en puerto 80]**

---

### 3.2. Zona Protegida con Autenticación Básica HTTP (1 punto)

Para restringir el acceso al directorio `/privado/` dentro de `web1.empresa.test`, se instaló el paquete `apache2-utils` para generar un fichero de credenciales cifradas mediante `htpasswd`:

```bash
sudo apt install apache2-utils -y
sudo htpasswd -c /etc/nginx/.htpasswd admin
mkdir -p /var/www/web1.empresa.test/html/privado
```

* **Seguridad de ubicación:** El fichero `.htpasswd` se almacenó en `/etc/nginx/` (fuera de `/var/www/`) para evitar que ningún usuario pueda descargarlo vía web.

Se añadió el bloque `location /privado/` dentro de `/etc/nginx/sites-available/web1.empresa.test`:

```nginx
    location /privado/ {
        auth_basic "Zona privada";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
```

#### Explicación de las directivas utilizadas:
* **`location /privado/`:** Aplica las reglas de seguridad exclusivamente a las peticiones dirigidas a esa ruta y sus subrutas.
* **`auth_basic "Zona privada";`:** Activa la autenticación básica HTTP y define el mensaje (*realm*) que muestra el navegador en la ventana emergente de inicio de sesión.
* **`auth_basic_user_file /etc/nginx/.htpasswd;`:** Indica la ruta del archivo donde Nginx verificará el usuario y el hash de la contraseña introducida.

#### Verificación realizada:
* Sin credenciales (`curl -i http://web1.empresa.test/privado/`): Devuelve **`HTTP/1.1 401 Unauthorized`**.
* Con credenciales válidas (`curl -i -u admin:**** http://web1.empresa.test/privado/`): Devuelve **`HTTP/1.1 200 OK`**.

> **[INSERTAR CAPTURA: Pruebas `curl -i` (401 y 200) y ventana emergente de login en Firefox]**

---

### 3.3. Web Dinámica con PHP y FastCGI (2 puntos)

Nginx no interpreta código PHP de forma nativa, por lo que delega su ejecución a un proceso externo llamado **PHP-FPM** (*FastCGI Process Manager*) comunicándose a través de un socket Unix.

```bash
sudo apt install php-fpm php-mysql -y
mkdir -p /var/www/php.empresa.test/html
```

Se creó `/var/www/php.empresa.test/html/index.php` mostrando la fecha/hora del servidor en tiempo real y la versión del motor PHP (`8.5.4`).

#### Configuración de `php.empresa.test` (`/etc/nginx/sites-available/php.empresa.test`)
```nginx
server {
    listen 80;
    listen [::]:80;

    server_name php.empresa.test;

    root /var/www/php.empresa.test/html;
    index index.php index.html;

    access_log /var/log/nginx/php.access.log;
    error_log  /var/log/nginx/php.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.5-fpm.sock;
    }
}
```

#### Explicación de las directivas utilizadas:
* **`index index.php index.html;`:** Prioriza el archivo `index.php` frente a `index.html` al acceder a la raíz del sitio.
* **`location ~ \.php$`:** Utiliza una expresión regular (`~`) para interceptar cualquier petición cuya URI termine en `.php` y enviarla al procesador PHP en lugar de entregar el código fuente como texto plano.
* **`include snippets/fastcgi-php.conf;`:** Carga los parámetros estándar de FastCGI (como `SCRIPT_FILENAME`) necesarios para que PHP-FPM sepa qué archivo ejecutar.
* **`fastcgi_pass unix:/run/php/php8.5-fpm.sock;`:** Especifica el socket local a través del cual Nginx envía la petición al servicio `php8.5-fpm`.

> **[INSERTAR CAPTURA: Fichero `php.empresa.test`, código `index.php` y navegador mostrando hora dinámica y PHP 8.5.4]**

---

### 3.4. Web Dinámica con PHP y Base de Datos MySQL (3 puntos)

Se instaló `mysql-server` y se creó una base de datos llamada `tienda`, junto con un usuario dedicado con permisos restringidos exclusivamente a dicha base de datos (siguiendo el principio de mínimo privilegio, evitando usar `root` desde la aplicación web):

```sql
CREATE DATABASE tienda CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'webuser'@'localhost' IDENTIFIED BY 'Web_Pass_2026!';
GRANT ALL PRIVILEGES ON tienda.* TO 'webuser'@'localhost';
FLUSH PRIVILEGES;

USE tienda;
CREATE TABLE productos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    precio DECIMAL(8,2) NOT NULL
);

INSERT INTO productos (nombre, precio) VALUES
    ('Teclado', 25.90),
    ('Ratón', 12.50),
    ('Monitor', 149.99);
```

#### Conexión desde PHP mediante PDO (`/var/www/bd.empresa.test/html/index.php`)
Se programó la página utilizando la extensión **PDO** (*PHP Data Objects*) con manejo de excepciones (`try/catch`), escape de salida HTML mediante `htmlspecialchars()` para prevenir inyección XSS y formato numérico local con `number_format()`.

```php
<?php
try {
    $pdo = new PDO("mysql:host=localhost;dbname=tienda;charset=utf8mb4", "webuser", "Web_Pass_2026!", [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION
    ]);
    $productos = $pdo->query("SELECT id, nombre, precio FROM productos");
} catch (PDOException $e) {
    die("Error de conexión: " . $e->getMessage());
}
?>
```

Se creó el *server block* `/etc/nginx/sites-available/bd.empresa.test` con la misma estructura FastCGI que `php.empresa.test`, se enlazó en `sites-enabled` y se recargaron Nginx y PHP-FPM.

> **[INSERTAR CAPTURA: Fichero `bd.empresa.test`, tabla `productos` en MySQL y página renderizada en Firefox]**

---

## 4. Incidencias Encontradas y Soluciones Aplicadas

Durante el despliegue e integración de los servicios surgieron las siguientes incidencias técnicas, documentadas junto con su diagnóstico y resolución:

| # | Incidencia / Error observado | Causa raíz | Solución aplicada |
| :--- | :--- | :--- | :--- |
| **1** | **Fallo de resolución DNS en el Router al usar Firefox:** El navegador del Router no resolvía `*.empresa.test`. | El archivo `/etc/resolv.conf` estaba gestionado dinámicamente por `systemd-resolved` y apuntaba al DNS del adaptador NAT (`192.168.111.1`) en lugar de al `bind9` local. | Se editó `/etc/systemd/resolved.conf` estableciendo `DNS=10.10.10.1` y se reinició el servicio con `sudo systemctl restart systemd-resolved`. |
| **2** | **`Connection refused` al conectar por SSH desde Windows al puerto `2222`:** Inicialmente se configuró el reenvío de puertos de VirtualBox directamente hacia `10.10.10.20:22`. | El motor NAT de VirtualBox solo puede reenviar puertos hacia las interfaces propias de la máquina donde está configurado (el Router), no hacia otras VMs de una red interna sin DNAT en `iptables`. | Se apuntó la regla de reenvío al propio Router (`127.0.0.1:2222` $\rightarrow$ `22`) y se conectó usando el Router como bastión mediante `ssh -J david@127.0.0.1:2222 david@10.10.10.20`. |
| **3** | **`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` al iniciar sesión SSH:** El cliente SSH de Windows bloqueó la conexión al puerto `2222`. | Existía una huella criptográfica antigua almacenada en `C:\Users\Alumno\.ssh\known_hosts` asociada a `[127.0.0.1]:2222` de una máquina virtual previa. | Se eliminó la clave obsoleta ejecutando `ssh-keygen -R "[127.0.0.1]:2222"` y se aceptó la nueva huella del servidor. |
| **4** | **`Unit php8.3-fpm.service not found` al reiniciar PHP-FPM:** El servicio falló al intentar reiniciarlo tras configurar Nginx. | Los repositorios instalados desplegaron la versión **PHP 8.5** (`/run/php/php8.5-fpm.sock`) en lugar de PHP 8.3. | Se verificó el socket activo con `ls /run/php/`, se actualizó la directiva `fastcgi_pass` a `unix:/run/php/php8.5-fpm.sock;` en `php.empresa.test` y `bd.empresa.test`, y se reinició `php8.5-fpm`. |
| **5** | **`-bash: syntax error near unexpected token '('` al probar `curl`:** Error de sintaxis al ejecutar la comprobación de la zona privada. | Al copiar la URL desde el navegador/editor se incluyó el formato de enlace Markdown `[url](url)` con paréntesis no escapados en la shell. | Se ejecutó el comando limpio sin caracteres de formato: `curl -i http://web1.empresa.test/privado/`. |

---

## 5. Conclusión y Comprobación Final

Se ha verificado mediante `sudo ss -tlnp | grep nginx` que todos los sitios web (`web1.empresa.test`, `web2.empresa.test`, `php.empresa.test` y `bd.empresa.test`) operan simultáneamente bajo el puerto `80` mediante *Virtual Hosting* por nombre, integrados completamente con el direccionamiento dinámico de **Kea DHCP** y la resolución de nombres de **bind9**.
