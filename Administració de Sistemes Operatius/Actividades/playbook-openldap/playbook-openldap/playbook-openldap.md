# Playbook de Despliegue Técnico: Servidor OpenLDAP (planetafp.local)

---

## 1. Objetivo

Instalar y configurar un servidor **OpenLDAP** en una máquina virtual con Ubuntu Server, usando una red interna con IP fija, y comprobar que el servicio funciona correctamente.

Al terminar el playbook se habrá conseguido:

- Una red NAT en VirtualBox sin DHCP.
- Un servidor Ubuntu con la IP fija `20.0.0.5`.
- El nombre `ldapserver.planetafp.local` resuelto de forma local.
- El servicio `slapd` instalado, activo y escuchando en el puerto 389.
- El directorio base `dc=planetafp,dc=local` creado y comprobado.

---

## 2. Ficha técnica del entorno

| Parámetro | Valor |
| --- | --- |
| Hipervisor | VirtualBox |
| Nombre de la máquina virtual | Ubuntu server - Playbook |
| Red de VirtualBox | Red NAT llamada `RedLDAP`, rango `20.0.0.0/24`, **DHCP deshabilitado** |
| Adaptador de red | Intel PRO/1000 MT Desktop, conectado a "Red NAT" |
| Sistema operativo | Ubuntu Server (la versión exacta se ve con `lsb_release -a`) |
| Usuario del sistema | `honeynet` |
| Nombre del equipo (hostname) | `ldapserver` |
| Interfaz de red | `enp0s3` |
| Dirección IP | `20.0.0.5/24` (estática, con Netplan) |
| Puerta de enlace | `20.0.0.1` |
| DNS externos | `8.8.8.8` y `1.1.1.1` |
| Nombre completo (FQDN) | `ldapserver.planetafp.local` |
| Dominio del directorio | `planetafp.local` → `dc=planetafp,dc=local` |
| Organización | `planetafp` |
| Administrador de LDAP | `cn=admin,dc=planetafp,dc=local` |
| Versión de OpenLDAP | slapd 2.6.10 (se ve en la captura 13) |
| Paquetes | `slapd`, `ldap-utils` |
| Puerto del servicio | 389/TCP |

---

## 3. Conceptos básicos

**Servicio de directorio:** es una base de datos pensada para **leer y buscar** información rápido. Guarda los datos en forma de árbol (jerarquía), llamado **DIT**. No sirve para operaciones complejas como una base de datos normal.

**LDAP:** (*Lightweight Directory Access Protocol*) es el protocolo que se usa para consultar y modificar un directorio. Funciona sobre TCP/IP, en el puerto 389.

**OpenLDAP:** es el programa libre que implementa LDAP.

**Usos típicos:** autenticación centralizada, gestión de usuarios y grupos, libretas de direcciones.

### Componentes

| Componente | Para qué sirve |
| --- | --- |
| `slapd` | Es el servidor de OpenLDAP. Recibe las peticiones, aplica los esquemas y guarda el directorio. |
| `ldap-utils` | Herramientas de consola para trabajar con LDAP: `ldapsearch`, `ldapadd`, `ldapmodify`, `ldapdelete`, etc. |
| `libldap` y `liblber` | Librerías que usan el servidor y las herramientas para entender el protocolo. |

---

## 4. Procedimiento de despliegue

### Fase 0. Preparar VirtualBox y la máquina virtual

#### Paso 0.1. Crear la red NAT

En VirtualBox: **Herramientas → Redes → Redes NAT → Crear**. Datos de la red:

| Campo | Valor |
| --- | --- |
| Nombre | `RedLDAP` |
| Prefijo IPv4 | `20.0.0.0/24` |
| Habilitar DHCP | **Desmarcado** |
| Habilitar IPv6 | Desmarcado |

**¿Por qué se desactiva el DHCP?** Porque el servidor tendrá una IP fija. Un servidor de directorio siempre debe estar en la misma dirección para que los clientes lo encuentren.

![Red NAT RedLDAP creada con DHCP inhabilitado](capturas/01-red-nat-virtualbox.png)

*Captura 1: la red `RedLDAP` aparece con prefijo `20.0.0.0/24` y servidor DHCP "Inhabilitado".*

#### Paso 0.2. Conectar la máquina virtual a la red

En la configuración de la máquina virtual, entrar en **Red → Adaptador 1** y poner:

- Habilitar adaptador de red: **activado**.
- Conectar a: **Red NAT**.
- Nombre: **RedLDAP**.

![Adaptador de red de la máquina virtual](capturas/02-adaptador-red-vm.png)

*Captura 2: el Adaptador 1 está conectado a la Red NAT `RedLDAP`.*

#### Paso 0.3. Instalar Ubuntu Server

Durante la instalación se crean el usuario y el nombre del servidor:

| Campo del instalador | Valor |
| --- | --- |
| Su nombre | `honeynet` |
| Nombre del servidor | `ldapserver` |
| Nombre de usuario | `honeynet` |
| Contraseña | *(contraseña elegida)* |

![Pantalla de usuario y nombre del servidor durante la instalación](capturas/03-instalacion-usuario-servidor.png)

*Captura 3: se ve el usuario `honeynet` y el nombre del servidor `ldapserver`.*

---

### Fase 1. Configurar la red estática con Netplan

#### Paso 1.1. Ver el nombre de la interfaz

```bash
ip -br a
```

- `ip -br a` enseña un resumen (`-br` = *brief*) de las interfaces de red y sus IPs.
- La interfaz de este servidor se llama `enp0s3`. Es la que se usa en el fichero YAML.

![Resultado de ip -br a con la interfaz enp0s3](capturas/04-ip-br-a-interfaz.png)

*Captura 4: la interfaz `enp0s3` aparece en estado `UP`.*

#### Paso 1.2. Editar el fichero de Netplan

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Contenido del fichero:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 20.0.0.5/24
      routes:
        - to: default
          via: 20.0.0.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Qué hace cada parte:

| Línea | Explicación |
| --- | --- |
| `version: 2` | Versión de la sintaxis de Netplan. |
| `enp0s3` | Interfaz que se va a configurar. |
| `dhcp4: false` | Desactiva el DHCP, porque la red no tiene servidor DHCP. |
| `addresses` | IP fija en formato CIDR. `/24` equivale a la máscara `255.255.255.0`. |
| `routes` (`to: default`, `via`) | Puerta de enlace. Sin ella no hay salida a Internet y `apt` falla. |
| `nameservers` | Servidores DNS para poder resolver los nombres de los repositorios. |

> **Importante:** el formato YAML depende de la **indentación**. Se usan espacios, nunca tabuladores.

![Fichero Netplan abierto en nano](capturas/05-netplan-yaml.png)

*Captura 5: el fichero `00-installer-config.yaml` con la IP `20.0.0.5/24`, la puerta de enlace `20.0.0.1` y los DNS.*

#### Paso 1.3. Probar y aplicar la configuración

```bash
sudo netplan try
sudo netplan apply
ip -br a
ping -c 3 8.8.8.8
```

- `netplan try`: prueba la configuración y la revierte sola si no se confirma. Así no nos quedamos sin red por un error.
- `netplan apply`: aplica la configuración de forma definitiva.
- `ip -br a`: comprueba que la IP está puesta.
- `ping -c 3 8.8.8.8`: envía 3 paquetes para comprobar que hay salida a Internet.

![Resultado de netplan try](capturas/06-netplan-try.png)

*Captura 6: `netplan try` termina con el mensaje "Configuration accepted".*

![Resultado de ip -br a con la IP 20.0.0.5/24](capturas/07-ip-br-a-final.png)

*Captura 7: la interfaz `enp0s3` tiene la IP `20.0.0.5/24`.*

![Ping a 8.8.8.8 correcto](capturas/08-ping-internet.png)

*Captura 8: se envían 3 paquetes y se reciben 3 (0 % de pérdida). Hay conexión a Internet.*

---

### Fase 2. Resolución de nombres local

#### Paso 2.1. Editar el fichero `/etc/hosts`

```bash
sudo nano /etc/hosts
```

Línea añadida:

```text
20.0.0.5   ldapserver.planetafp.local   ldapserver
```

- Formato de la línea: `IP  nombre-completo  alias`.

**¿Por qué se hace?** En la red del laboratorio no hay servidor DNS. Con esta línea, el equipo sabe que `ldapserver.planetafp.local` y `ldapserver` son la IP `20.0.0.5`.

> Esta línea solo funciona en la máquina donde se escribe. Si más adelante se añade un cliente LDAP, ese cliente necesita la misma línea.

![Fichero /etc/hosts con la línea añadida](capturas/09-etc-hosts.png)

*Captura 9: aparece la línea `20.0.0.5 ldapserver.planetafp.local ldapserver`.*

#### Paso 2.2. Comprobar la resolución

```bash
getent hosts ldapserver.planetafp.local
```

- `getent hosts` pregunta al sistema cómo resuelve un nombre (incluye el fichero `/etc/hosts`).

![Resultado de getent hosts](capturas/10-getent-hosts.png)

*Captura 10: el nombre `ldapserver.planetafp.local` devuelve la IP `20.0.0.5`.*

---

### Fase 3. Instalación del software

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install slapd ldap-utils -y
```

- `apt update`: actualiza la lista de paquetes disponibles.
- `&&`: ejecuta el siguiente comando solo si el anterior terminó bien.
- `apt upgrade -y`: actualiza los paquetes instalados (`-y` contesta "sí" automáticamente).
- `apt install slapd ldap-utils -y`: instala el servidor LDAP y sus herramientas.

Durante la instalación pide una contraseña de administrador. Se vuelve a configurar en la Fase 4.

---

### Fase 4. Reconfiguración de slapd

```bash
sudo dpkg-reconfigure slapd
```

- `dpkg-reconfigure` vuelve a abrir el asistente de configuración de un paquete ya instalado.
- Es necesario porque la instalación crea un directorio genérico (`nodomain`) que no sirve para nuestra organización.

Respuestas en cada pantalla:

| Pregunta del asistente | Respuesta |
| --- | --- |
| ¿Omitir la configuración del servidor OpenLDAP? | **No** |
| Nombre de dominio DNS | `planetafp.local` (crea la base `dc=planetafp,dc=local`) |
| Nombre de la organización | `planetafp` |
| Contraseña del administrador | *(contraseña elegida)*, y se repite para confirmar |
| ¿Borrar la base de datos al purgar slapd? | **Yes** (si se desinstala con `apt purge`, se borra también la base de datos) |
| ¿Mover la base de datos antigua? | **Yes** (guarda la base `nodomain` como copia de seguridad) |

![Pantalla de nombre de dominio DNS](capturas/11-slapd-dominio.png)

*Captura 11: en el asistente se escribe el dominio `planetafp.local`.*

![Pantalla de nombre de la organización](capturas/12-slapd-organizacion.png)

*Captura 12: en el asistente se escribe la organización `planetafp`.*

---

## 5. Verificación

### 5.1. Comprobar que el servicio está activo

```bash
sudo systemctl status slapd
sudo ss -tlnp | grep slapd
```

- `systemctl status slapd`: muestra el estado del servicio. Debe poner `active (running)`.
- `ss -tlnp`: lista los puertos TCP en escucha (`-t` TCP, `-l` escuchando, `-n` números, `-p` proceso).
- `grep slapd`: deja solo las líneas del servicio `slapd`. Debe aparecer el puerto **389**.

![Estado del servicio slapd](capturas/13-slapd-status.png)

*Captura 13: `slapd.service` aparece como `active (running)` y habilitado (`enabled`). Es la versión 2.6.10.*

![slapd escuchando en el puerto 389](capturas/14-slapd-puerto-389.png)

*Captura 14: `slapd` escucha en el puerto `389`, tanto en IPv4 (`0.0.0.0:389`) como en IPv6 (`[::]:389`).*

### 5.2. Comprobar los esquemas

```bash
dpkg -L slapd | grep schema
ls /etc/ldap/schema/
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=schema,cn=config" dn
```

- `dpkg -L slapd`: lista los ficheros que instala el paquete. `grep schema` deja solo los de esquemas.
- `ls /etc/ldap/schema/`: muestra los ficheros de esquemas. En Ubuntu y Debian la ruta es esta.
- `ldapsearch -Y EXTERNAL -H ldapi:/// ...`: consulta la configuración interna de LDAP como `root`, usando el socket local. Muestra los esquemas cargados.

Los esquemas cargados son:

| Esquema | Para qué sirve |
| --- | --- |
| `core` | Definiciones básicas de LDAP. |
| `cosine` | Atributos y clases de uso común (X.500). |
| `nis` | Datos de usuarios y grupos tipo Unix. |
| `inetorgperson` | Clase `inetOrgPerson`, para personas. |

![Esquemas cargados en el servidor](capturas/15-esquemas-config.png)

*Captura 15: aparecen los 4 esquemas (`core`, `cosine`, `nis`, `inetorgperson`). El resultado es `0 Success` y `numEntries: 5`.*

### 5.3. Comprobar el directorio base

```bash
ldapsearch -x -H ldap://ldapserver.planetafp.local -b "dc=planetafp,dc=local" -s base
```

- `-x`: autenticación simple (aquí es anónima).
- `-H`: dirección del servidor.
- `-b`: entrada desde donde empieza la búsqueda.
- `-s base`: mira solo esa entrada, sin buscar por debajo.

![Consulta de la base del directorio](capturas/16-ldapsearch-base.png)

*Captura 16: se obtiene la entrada `dc=planetafp,dc=local` con `o: planetafp`. El resultado es `0 Success` y `numEntries: 1`. Además, el nombre `ldapserver.planetafp.local` funciona.*

---

## 6. Poblar el directorio (operaciones básicas)

### 6.1. Crear el fichero LDIF

El fichero `estructura.ldif` define una unidad organizativa (`ou=alumno`) y un usuario dentro de ella.

```bash
nano estructura.ldif
```

Contenido **correcto**:

```text
dn: ou=alumno,dc=planetafp,dc=local
objectClass: organizationalUnit
ou: alumno

dn: uid=jperez,ou=alumno,dc=planetafp,dc=local
objectClass: inetOrgPerson
cn: Juan Perez
sn: Perez
uid: jperez
mail: jperez@planetafp.local
```

Reglas del formato LDIF:

- Cada entrada empieza con `dn:` (nombre completo de la entrada). Después van los atributos como `atributo: valor`.
- Entre una entrada y otra va **una línea en blanco**.
- El **padre debe existir antes que el hijo**. Por eso la unidad `ou=alumno` va primero.
- La clase `inetOrgPerson` obliga a poner `cn` y `sn`.
- No se usan tildes en los valores. LDIF exige codificarlas en base64 y es un error muy común.

![Fichero estructura.ldif abierto en nano](capturas/17-estructura-ldif.png)

*Captura 17: el fichero `estructura.ldif` en nano.*

> **Error detectado en la captura 17:** en la segunda entrada se escribió `dc=planeta,dc=local`. Lo correcto es `dc=planetafp,dc=local`, como en el bloque de arriba. Con el error, `ldapadd` daría `No such object (32)`, porque el padre indicado no existe. Hay que corregirlo antes de cargar el fichero.

### 6.2. Cargar los datos con `ldapadd`

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -H ldap://localhost -f estructura.ldif
```

- `ldapadd`: añade entradas al directorio.
- `-x`: autenticación simple (usuario y contraseña).
- `-D`: usuario con el que se conecta (el administrador).
- `-W`: pide la contraseña por teclado, para que no quede en el historial.
- `-H ldap://localhost`: servidor al que se conecta.
- `-f`: fichero LDIF que se carga.

Salida esperada:

```text
Enter LDAP Password:
adding new entry "ou=alumno,dc=planetafp,dc=local"

adding new entry "uid=jperez,ou=alumno,dc=planetafp,dc=local"
```

*(Pendiente: añadir captura de este paso cuando se ejecute.)*

### 6.3. Consultar los datos con `ldapsearch`

```bash
ldapsearch -x -H ldap://localhost -b "ou=alumno,dc=planetafp,dc=local" -s sub "(uid=jperez)" uid cn mail
```

- `-b`: dónde empieza la búsqueda.
- `-s sub`: busca en la base y en todo lo que cuelga de ella. Otras opciones son `base` (solo la base) y `one` (solo los hijos directos).
- `"(uid=jperez)"`: filtro de búsqueda, entre comillas para que la consola no interprete los paréntesis.
- `uid cn mail`: atributos que se quieren ver. Sin ellos se muestran todos.

Salida esperada:

```text
dn: uid=jperez,ou=alumno,dc=planetafp,dc=local
uid: jperez
cn: Juan Perez
mail: jperez@planetafp.local

# numEntries: 1
```

*(Pendiente: añadir captura de este paso cuando se ejecute.)*

---

## 7. Solución de problemas (Troubleshooting)

| Nº | Síntoma o error | Causa probable | Solución |
| --- | --- | --- | --- |
| 1 | `Invalid syntax (21)` o `Object class violation (65)` | Error en el LDIF: falta una línea en blanco, falta un atributo obligatorio (`cn`, `sn`), clase mal escrita o tildes. | Revisar el fichero línea a línea. Usar `cat -A estructura.ldif` para ver espacios y saltos de línea ocultos. |
| 2 | `No such object (32)` | Se crea una entrada cuyo padre no existe. También ocurre si el dominio del `dn` está mal escrito (por ejemplo `dc=planeta` en vez de `dc=planetafp`). | Poner los padres primero. Revisar que el dominio sea `dc=planetafp,dc=local`. |
| 3 | `Already exists (68)` | Ya existe una entrada con ese `dn`. | Usar otro `dn`, o borrar la entrada con `ldapdelete`, o cambiarla con `ldapmodify`. |
| 4 | `Invalid credentials (49)` | Contraseña incorrecta o `dn` del administrador mal escrito. | Revisar el valor de `-D`. Si se olvidó la contraseña, se puede ejecutar `sudo dpkg-reconfigure slapd` (esto recrea la base y se pierden los datos). |
| 5 | `Can't contact LDAP server (-1)` | `slapd` está parado, la URL es incorrecta o el puerto no responde. | Ejecutar `sudo systemctl status slapd` y `sudo systemctl restart slapd`. Comprobar el puerto con `sudo ss -tlnp \| grep 389`. Revisar el valor de `-H`. |
| 6 | `Name or service not known` al usar `ldapserver.planetafp.local` | Falta la línea en `/etc/hosts` o está mal escrita. | Revisar `/etc/hosts` y probar con `getent hosts ldapserver.planetafp.local`. |
| 7 | `apt update` falla con `Temporary failure resolving` | Falta la puerta de enlace o los DNS en Netplan. | Revisar `routes` y `nameservers` en el YAML, ejecutar `sudo netplan apply` y probar `ping 8.8.8.8`. |
| 8 | `netplan apply` da errores de YAML | Indentación incorrecta o tabuladores. | Usar solo espacios. Ejecutar `sudo netplan generate` y corregir la línea que indique. |

---

## 8. Lista de comprobación final

Marca cada punto cuando esté hecho:

- [x] Red NAT `RedLDAP` creada con `20.0.0.0/24` y sin DHCP (captura 1).
- [x] Adaptador de la máquina virtual conectado a `RedLDAP` (captura 2).
- [x] Ubuntu Server instalado con usuario `honeynet` y servidor `ldapserver` (captura 3).
- [x] IP fija `20.0.0.5/24` aplicada con Netplan (capturas 4 a 7).
- [x] Salida a Internet comprobada con `ping` (captura 8).
- [x] Línea añadida en `/etc/hosts` y resolución comprobada (capturas 9 y 10).
- [x] `slapd` y `ldap-utils` instalados.
- [x] `slapd` reconfigurado con `planetafp.local` y `planetafp` (capturas 11 y 12).
- [x] Servicio `slapd` activo y escuchando en el puerto 389 (capturas 13 y 14).
- [x] Esquemas `core`, `cosine`, `nis` e `inetorgperson` cargados (captura 15).
- [x] Directorio base `dc=planetafp,dc=local` comprobado (captura 16).
- [ ] Fichero `estructura.ldif` corregido (`dc=planetafp`, no `dc=planeta`) (captura 17).
- [ ] Datos cargados con `ldapadd` y consultados con `ldapsearch`.

---

*Playbook elaborado como guía de despliegue de OpenLDAP en un entorno de laboratorio.*
