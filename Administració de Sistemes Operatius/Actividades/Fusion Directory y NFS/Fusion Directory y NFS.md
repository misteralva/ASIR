# Manual de Operaciones: Gestión de Usuarios y Almacenamiento Centralizado (Honeynet)

**Proyecto:** Honeynet (Red Cebo y Análisis de Ciberseguridad)  
**Dominio LDAP:** `dc=honeynet,dc=local`  
**Carpeta Compartida:** `/srv/nfs/telemetria`  
**Grupo de Trabajo:** `analistas`  

---

## 1. ¿Qué queremos conseguir con esta práctica?

En nuestro proyecto Honeynet necesitamos organizar el acceso de los miembros del equipo y guardar de forma segura los archivos de registro (logs) que generan los sensores de ciberseguridad.

Para lograrlo, conectaremos tres herramientas principales:
1. **OpenLDAP:** Funciona como una base de datos centralizada donde se guardan las cuentas de los usuarios y grupos.
2. **Fusion Directory:** Es un panel web sencillo que nos permite crear y administrar usuarios y grupos usando botones, sin necesidad de comandos complejos.
3. **Servidor NFS:** Es una carpeta compartida en la red (`/srv/nfs/telemetria`) donde los analistas pueden guardar y leer datos de seguridad.

---

## 2. Requisitos previos

Antes de empezar, debemos asegurarnos de tener listos los siguientes puntos en nuestra máquina o servidor:

* El servicio de **OpenLDAP** ya está funcionando con el dominio `dc=honeynet,dc=local`.
* El sistema operativo reconoce los usuarios y grupos de LDAP a través de la herramienta `nslcd` (esto se comprueba ejecutando `getent passwd` y `getent group` en la terminal).
* Contamos con una cuenta de administrador de LDAP (por ejemplo: `cn=admin,dc=honeynet,dc=local`).

---

## 3. Módulo A: Instalación y Configuración de Fusion Directory

### Paso 1: Instalar programas necesarios
En el servidor donde instalaremos el panel web, ejecutamos los siguientes comandos para preparar la web (Apache y PHP) y añadir la lista oficial de programas de Fusion Directory:

```bash
# Actualizamos la lista de paquetes e instalamos el servidor web
sudo apt update
sudo apt install -y apache2 php php-ldap php-mbstring php-xml schema2ldif

# Descargamos la clave de seguridad del programa
wget -qO - https://repos.fusiondirectory.org/fusiondirectory.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/fusiondirectory.gpg

# Añadimos el repositorio oficial
echo "deb http://repos.fusiondirectory.org/fusiondirectory-1.3/debian/bookworm bookworm main" | sudo tee /etc/apt/sources.list.d/fusiondirectory.list

# Instalamos Fusion Directory
sudo apt update
sudo apt install -y fusiondirectory fusiondirectory-schema
```

> **Nota importante:** Solo instalamos el paquete básico. No instalamos extensiones extra que no necesitemos para mantener el sistema ligero y fácil de administrar.

---

### Paso 2: Añadir las plantillas (esquemas) a OpenLDAP
Para que OpenLDAP entienda la información que le enviará el panel web, cargamos estas reglas básicas mediante el siguiente comando:

```bash
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/core-fd.schema
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/core-fd-conf.schema
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/ldapns.schema
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/template-fd.schema
```

---

### Paso 3: Asistente de configuración Web paso a paso
Abrimos un navegador web e ingresamos a la dirección: `http://<IP_DE_TU_SERVIDOR>/fusiondirectory/`.

Seguimos las 6 pantallas del asistente:

1. **Bienvenida (Token):** El sistema nos pide una clave de seguridad por protección. La creamos ejecutando este comando en la terminal:
   ```bash
   echo "clave123" | sudo tee /var/spool/fusiondirectory/setup_token
   ```
2. **Idioma:** Elegimos el idioma (Español o Inglés).
3. **Comprobación:** El sistema verifica que todo lo necesario esté instalado correctamente (debe aparecer todo en verde).
4. **Conexión a LDAP:** Rellenamos los datos de nuestra base de datos:
   * **Base DN:** `dc=honeynet,dc=local`
   * **Admin DN:** `cn=admin,dc=honeynet,dc=local`
   * **Contraseña:** Ponemos la contraseña de nuestro administrador LDAP.
5. **Estructura de carpetas:** Confirmamos la creación de dos carpetas principales en el directorio:
   * `ou=usuarios` (para guardar las cuentas de personas).
   * `ou=grupos` (para guardar los equipos de trabajo).
6. **Finalizar:** Descargamos o guardamos el archivo de configuración final en `/etc/fusiondirectory/fusiondirectory.conf`.

---

### Paso 4: Crear Grupos y Usuarios desde la web
Entramos al panel web con la cuenta de administrador y creamos nuestro equipo de trabajo:

1. **Crear el Grupo `analistas`:**
   * Vamos al menú **Grupos** -> **Añadir**.
   * Le ponemos de nombre `analistas` y activamos la opción de **Grupo POSIX** para que Linux lo reconozca.
2. **Crear un Usuario Analista:**
   * Vamos al menú **Usuarios** -> **Añadir**.
   * Rellenamos el nombre, apellido y correo del usuario.
   * En la pestaña **POSIX**, le asignamos una contraseña, la carpeta personal (ejemplo: `/home/analista1`) y lo añadimos dentro del grupo `analistas`.

---

## 4. Módulo B: Compartir la Carpeta de Red (Servidor NFS)

Ahora configuraremos una carpeta central en el servidor para almacenar las capturas de tráfico y archivos de registro (logs) de nuestra Honeynet.

### Paso 1: Instalar el servicio NFS y crear la carpeta
```bash
# Instalamos el programa para compartir carpetas en Linux
sudo apt install -y nfs-kernel-server

# Creamos la carpeta donde guardaremos los archivos de seguridad
sudo mkdir -p /srv/nfs/telemetria
```

---

### Paso 2: Dar permisos solo al grupo de analistas
Para garantizar que solo el personal autorizado pueda entrar a la carpeta, asignamos la propiedad al grupo de LDAP `analistas` y bloqueamos el acceso a cualquier otra persona:

```bash
# Le entregamos la carpeta al usuario root y al grupo analistas
sudo chown root:analistas /srv/nfs/telemetria

# Permiso 770: El dueño y los analistas pueden leer, escribir y entrar. El resto del mundo no puede hacer nada.
sudo chmod 770 /srv/nfs/telemetria
```

---

### Paso 3: Configurar el archivo de red `/etc/exports`
Editamos el archivo `/etc/exports` para autorizar a los equipos de nuestra red local a conectarse a la carpeta compartida:

```text
/srv/nfs/telemetria 192.168.1.0/24(rw,sync,no_root_squash)
```

Aplicamos los cambios para activar la carpeta compartida:
```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

---

### Paso 4: Conectar la carpeta desde un equipo cliente
En la máquina o equipo del analista, instalamos el cliente NFS y conectamos la carpeta:

```bash
# Instalamos las herramientas de conexión
sudo apt install -y nfs-common
sudo mkdir -p /mnt/nfs/telemetria

# Conectamos la carpeta compartida (reemplaza la IP por la de tu servidor)
sudo mount 192.168.1.5:/srv/nfs/telemetria /mnt/nfs/telemetria
```

Para que la carpeta siga conectada automáticamente cuando reiniciemos la máquina, añadimos esta línea al final del archivo `/etc/fstab`:

```text
192.168.1.5:/srv/nfs/telemetria /mnt/nfs/telemetria nfs defaults 0 0
```

---

## 5. Módulo C: Comprobación y Verificación del Funcionamiento

### Comandos útiles de comprobación

* **En el Servidor:**
  ```bash
  # Ver qué carpetas estamos compartiendo en la red
  sudo exportfs -v

  # Comprobar quién es el dueño y qué permisos tiene la carpeta
  ls -ld /srv/nfs/telemetria
  # La respuesta debe ser: drwxrwx--- ... root analistas /srv/nfs/telemetria
  ```

* **En la Máquina Cliente:**
  ```bash
  # Ver si el servidor nos permite conectarnos
  showmount -e 192.168.1.5

  # Confirmar que la carpeta está montada y ver cuánto espacio libre tiene
  df -h | grep nfs
  ```

---

### Prueba práctica de seguridad (Demostración)

#### Prueba A: Intentar entrar con un usuario normal (No autorizado)
Iniciamos sesión con un usuario local que **no** pertenece al grupo `analistas` (por ejemplo, la cuenta `ubuntu`):

```bash
ubuntu@cliente:~$ ls /mnt/nfs/telemetria
ls: cannot open directory '/mnt/nfs/telemetria': Permission denied
```
* **Resultado:** **Acceso denegado.** El sistema bloquea al usuario porque no forma parte del grupo `analistas`.

#### Prueba B: Entrar con un Analista de LDAP (Autorizado)
Iniciamos sesión con un usuario creado en Fusion Directory que **sí** pertenece al grupo `analistas`:

```bash
analista1@cliente:~$ cd /mnt/nfs/telemetria
analista1@cliente:/mnt/nfs/telemetria$ touch log_ataque.json
analista1@cliente:/mnt/nfs/telemetria$ ls -l
-rw-r--r-- 1 analista1 analistas 0 Oct 23 17:40 log_ataque.json
```
* **Resultado:** **Acceso concedido.** El analista puede ver la carpeta y crear archivos de registro sin ningún problema.

---

## Conclusión
La integración se ha completado con éxito. Gracias a esta configuración, el equipo de ciberseguridad puede gestionar los usuarios de manera sencilla usando el panel web de **Fusion Directory** y almacenar de forma segura los eventos del proyecto **Honeynet** en la carpeta compartida **NFS**.
