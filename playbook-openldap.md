# Playbook de Despliegue Técnico: OpenLDAP (planetafp.local)

> **Nota de uso:** donde aparece `[CAPTURA]` debes pegar tu propia captura o volcado de terminal obtenido al realizar la práctica. Los bloques "Salida esperada" indican cómo debería verse, pero pueden variar ligeramente (versiones, fechas, nombre de interfaz).

---

## A. Ficha Técnica y Prerrequisitos del Entorno

### A.1 Tabla resumen

| Parámetro | Valor |
|---|---|
| Hipervisor | VirtualBox |
| Topología de red | Red NAT (*NAT Network*) de VirtualBox, **sin DHCP**, rango 20.0.0.0/24 |
| Sistema operativo | Ubuntu Server (indicar versión exacta: `lsb_release -a`) |
| Dirección IP | 20.0.0.5/24 (estática, Netplan) |
| FQDN | ldapserver.planetafp.local |
| Alias corto | ldapserver |
| Dominio DNS / DIT | planetafp.local → `dc=planetafp,dc=local` |
| Organización | planetafp |
| DN del administrador | `cn=admin,dc=planetafp,dc=local` |
| Paquetes | `slapd`, `ldap-utils` (y dependencias `libldap`, `liblber`) |

### A.2 Descripción del servicio

Un **servicio de directorio** es una base de datos especializada en **lecturas y búsquedas rápidas** de información descriptiva organizada en forma de atributos y en estructura **jerárquica** (árbol, el *DIT*). No está pensado para transacciones complejas como un SGBD relacional. **LDAP** (*Lightweight Directory Access Protocol*) es el protocolo ligero, sobre TCP/IP (puerto 389), para consultar y modificar esos directorios; **OpenLDAP** es su implementación libre. Usos típicos: autenticación centralizada, gestión de usuarios y grupos, libretas de direcciones.

### A.3 Componentes principales

| Componente | Función |
|---|---|
| **slapd** | El *daemon* (servidor) OpenLDAP. Escucha peticiones, aplica esquemas y permisos, y guarda el directorio. |
| **ldap-utils** | Herramientas cliente de línea de comandos: `ldapsearch`, `ldapadd`, `ldapmodify`, `ldapdelete`, `ldapwhoami`, etc. |
| **libldap / liblber** | Bibliotecas que implementan el protocolo LDAP (`libldap`) y la codificación BER de los mensajes (`liblber`). Las usan servidor y clientes. |

---

## B. Procedimiento de Despliegue Paso a Paso

### Paso 1. Configuración de red estática (Netplan)

**1.1 Identificar el nombre de la interfaz:**

```bash
ip -br a
```

- `ip -br a` muestra de forma resumida (`-br` = *brief*) las interfaces y sus IPs. En VirtualBox suele llamarse `enp0s3`. Usa en el YAML el nombre que veas tú.

**1.2 Editar el fichero de Netplan** (el nombre puede variar, p. ej. `00-installer-config.yaml` o `50-cloud-init.yaml`):

```bash
ls /etc/netplan/
sudo nano /etc/netplan/00-installer-config.yaml
```

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

Explicación de cada parte:

- `version: 2`: versión de la sintaxis de Netplan.
- `enp0s3`: interfaz que se configura.
- `dhcp4: false`: desactiva DHCP, porque la red NAT no tiene servidor DHCP.
- `addresses`: IP estática en notación CIDR (`/24` = máscara 255.255.255.0).
- `routes` (`to: default`, `via`): puerta de enlace. **Ajusta `20.0.0.1` a la puerta de enlace real de tu Red NAT**; sin ella no habrá salida a Internet y `apt` fallará en el Paso 3.
- `nameservers`: DNS externos para que `apt` resuelva los repositorios.

> YAML es sensible a la **indentación** (espacios, nunca tabuladores).

**1.3 Aplicar y comprobar:**

```bash
sudo netplan try
sudo netplan apply
ip -br a
ping -c 3 8.8.8.8
```

- `netplan try`: aplica la configuración con reversión automática si no confirmas (evita quedarte sin red).
- `netplan apply`: aplica la configuración definitivamente.
- `ping -c 3 8.8.8.8`: envía 3 paquetes para comprobar la salida a Internet.

`[CAPTURA: ip -br a mostrando 20.0.0.5/24 y ping correcto]`

### Paso 2. Resolución de nombres local

```bash
sudo nano /etc/hosts
```

Línea añadida:

```text
20.0.0.5   ldapserver.planetafp.local   ldapserver
```

- Formato: `IP  FQDN  alias`. Primero va el nombre completo y luego los alias.

**Justificación:** no hay servidor DNS en la red del laboratorio. Con esta entrada la máquina resuelve `ldapserver.planetafp.local` y `ldapserver` sin depender de DNS ni de recordar la IP. El FQDN también debe coincidir con el dominio del directorio. **Solo afecta a la máquina donde se edita**: cualquier cliente LDAP adicional necesitará la misma línea (o un DNS que resuelva el nombre).

Comprobación:

```bash
getent hosts ldapserver.planetafp.local
```

- `getent hosts` consulta el resolvedor del sistema (incluyendo `/etc/hosts`). Debe devolver `20.0.0.5`.

> **Atención a la guía de apoyo:** el PDF usa `ldapserver.ifp.local` en el ejemplo y en la captura del menú, pero el enunciado exige `planetafp.local`. Usa siempre `planetafp.local`.

### Paso 3. Instalación de software

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install slapd ldap-utils -y
```

- `apt update`: refresca el índice de paquetes disponibles.
- `&&`: ejecuta el siguiente comando solo si el anterior terminó sin errores.
- `apt upgrade -y`: actualiza los paquetes instalados (`-y` responde "sí" automáticamente).
- `apt install slapd ldap-utils -y`: instala el servidor y las herramientas cliente.

> **Ojo:** el PDF escribe `sudo apt install update && sudo apt install upgrade`, que es incorrecto (`update` y `upgrade` son subcomandos de `apt`, no paquetes). Lo correcto es el bloque de arriba.

Durante la instalación se pide una **contraseña de administrador** (se pide dos veces). Se volverá a definir en el Paso 4.

`[CAPTURA: instalación finalizada]`

### Paso 4. Reconfiguración del servicio

```bash
sudo dpkg-reconfigure slapd
```

- `dpkg-reconfigure` vuelve a ejecutar el asistente de configuración de un paquete ya instalado. La instalación por defecto crea un directorio genérico (`nodomain`) que no sirve para nuestra organización.

Respuestas dadas en cada pantalla:

| Pregunta del asistente | Respuesta |
|---|---|
| Omit OpenLDAP server configuration? | **No** (queremos configurarlo) |
| DNS domain name | `planetafp.local` (genera el DN base `dc=planetafp,dc=local`) |
| Organization name | `planetafp` |
| Administrator password | *(contraseña elegida)* y repetirla en *Confirm password* |
| Do you want the database to be removed when slapd is purged? | **Yes** (si se purga el paquete con `apt purge`, se borra también la base de datos). Alternativa: *No* si se quiere conservar la información tras purgar. Indicar cuál se eligió y por qué. |
| Move old database? | **Yes** (mueve la base `nodomain` anterior a una copia de seguridad en lugar de bloquear la creación de la nueva) |

`[CAPTURA: pantallas del asistente con domain name y organization name]`

> Nota de seguridad: no escribas la contraseña real en este documento.

---

## C. Verificación y Comprobación

### C.1 Estado del daemon

```bash
sudo systemctl status slapd
sudo ss -tlnp | grep slapd
```

- `systemctl status slapd`: muestra si el servicio está activo (`active (running)`).
- `ss -tlnp`: lista sockets TCP (`-t`) en escucha (`-l`), en formato numérico (`-n`) y con el proceso (`-p`). `grep slapd` filtra solo nuestro servicio. Debe aparecer el puerto **389**.

Salida esperada (resumida):

```text
● slapd.service - LSB: OpenLDAP standalone server (Lightweight Directory Access Protocol)
     Active: active (running) since ...
LISTEN 0  2048  0.0.0.0:389  0.0.0.0:*  users:(("slapd",pid=...,fd=...))
```

`[CAPTURA]`

### C.2 Presencia de los esquemas

```bash
dpkg -L slapd | grep schema
ls /etc/ldap/schema/
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=schema,cn=config" dn
```

- `dpkg -L slapd`: lista los ficheros instalados por el paquete; `grep schema` deja solo los de esquemas.
- `ls /etc/ldap/schema/`: muestra los ficheros `.schema`/`.ldif` (`core`, `cosine`, `nis`, `inetorgperson`...). En Ubuntu/Debian la ruta es `/etc/ldap/schema/` (el PDF cita `/etc/openldap/schema/`, que es la de Red Hat).
- La tercera orden consulta la configuración dinámica (`cn=config`) como `root` mediante el socket local (`-Y EXTERNAL`, `-H ldapi:///`) y lista los esquemas cargados en el servidor.

Salida esperada (resumida):

```text
dn: cn=schema,cn=config
dn: cn={0}core,cn=schema,cn=config
dn: cn={1}cosine,cn=schema,cn=config
dn: cn={2}nis,cn=schema,cn=config
dn: cn={3}inetorgperson,cn=schema,cn=config
```

`[CAPTURA]`

### C.3 Comprobar el DIT base

```bash
ldapsearch -x -H ldap://ldapserver.planetafp.local -b "dc=planetafp,dc=local" -s base
```

- `-x`: autenticación simple (anónima aquí). `-H`: URL del servidor. `-b`: entrada de partida. `-s base`: solo examina esa entrada.

Salida esperada (resumida):

```text
dn: dc=planetafp,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: planetafp
dc: planetafp
# numEntries: 1
```

`[CAPTURA]`

---

## D. Población del Directorio y Operaciones Básicas

### D.1 Fichero LDIF de prueba (`estructura.ldif`)

```ldif
dn: ou=people,dc=planetafp,dc=local
objectClass: organizationalUnit
ou: people

dn: uid=jperez,ou=people,dc=planetafp,dc=local
objectClass: inetOrgPerson
cn: Juan Perez
sn: Perez
uid: jperez
mail: jperez@planetafp.local
```

Claves del formato:

- Cada entrada empieza por `dn:` (nombre distinguido completo) y sus atributos van como `atributo: valor`.
- **Una línea en blanco separa las entradas.**
- El **padre debe existir antes que el hijo**: por eso la OU va primero.
- `objectClass: inetOrgPerson` exige `cn` y `sn` (obligatorios) y permite `uid`, `mail`...
- He evitado tildes en los valores: el estándar LDIF pide codificar en base64 los caracteres no ASCII, y es un fallo frecuente.

### D.2 Inserción (`ldapadd`)

```bash
ldapadd -x -D "cn=admin,dc=planetafp,dc=local" -W -H ldap://localhost -f estructura.ldif
```

- `ldapadd`: añade entradas.
- `-x`: autenticación simple (usuario y contraseña, sin SASL).
- `-D`: DN con el que se hace el *bind* (el administrador).
- `-W`: pide la contraseña por teclado (así no queda en el historial).
- `-H ldap://localhost`: servidor al que conectar.
- `-f`: fichero LDIF a cargar.

Salida esperada:

```text
Enter LDAP Password:
adding new entry "ou=people,dc=planetafp,dc=local"

adding new entry "uid=jperez,ou=people,dc=planetafp,dc=local"
```

`[CAPTURA]`

### D.3 Consulta (`ldapsearch`)

```bash
ldapsearch -x -H ldap://localhost -b "ou=people,dc=planetafp,dc=local" -s sub "(uid=jperez)" uid cn mail
```

- `-b`: base de la búsqueda (solo se busca por debajo de `ou=people`).
- `-s sub`: ámbito. Opciones: `base` (solo la base), `one` (hijos inmediatos), `sub` (la base y todo lo que cuelga). Ojo: el valor se escribe `sub`, no `subtree`.
- `"(uid=jperez)"`: filtro; entre comillas para que la shell no interprete los paréntesis.
- `uid cn mail`: atributos a devolver. Sin ellos devuelve todos.

Salida esperada:

```text
dn: uid=jperez,ou=people,dc=planetafp,dc=local
uid: jperez
cn: Juan Perez
mail: jperez@planetafp.local

# numEntries: 1
```

Otra consulta útil, por clase de objeto:

```bash
ldapsearch -x -H ldap://localhost -b "dc=planetafp,dc=local" -s sub "(objectClass=inetOrgPerson)" dn
```

`[CAPTURA]`

---

## E. Control de Errores y Troubleshooting

| # | Síntoma / error | Causa probable | Solución |
|---|---|---|---|
| 1 | `ldap_add: Invalid syntax (21)` / `ldif_read_file: ... line N` / `Object class violation (65)` | Error en el LDIF: falta línea en blanco entre entradas, espacio antes del atributo, falta un atributo obligatorio (`sn`, `cn`), `objectClass` mal escrito o tildes sin codificar. | Revisar el fichero línea a línea. Verificar que cada `objectClass` existe en los esquemas cargados y que están todos los atributos obligatorios. Comprobar con `cat -A estructura.ldif` (muestra espacios finales y saltos de línea ocultos). |
| 2 | `ldap_add: No such object (32)` | Se intenta crear una entrada cuyo padre no existe (p. ej. el usuario antes que `ou=people`). | Ordenar el LDIF: padres primero, hijos después. Comprobar que el sufijo `dc=planetafp,dc=local` es el correcto. |
| 3 | `ldap_add: Already exists (68)` | Ya existe una entrada con ese DN. | Usar otro DN, o borrarla con `ldapdelete` o modificarla con `ldapmodify`. |
| 4 | `ldap_bind: Invalid credentials (49)` | Contraseña incorrecta o DN de administrador mal escrito (p. ej. `dc=ifp,dc=local` en lugar de `dc=planetafp,dc=local`). | Revisar el DN en `-D`. Si se olvidó la contraseña: `sudo dpkg-reconfigure slapd` (recrea la base de datos, perdiendo datos). |
| 5 | `ldap_sasl_bind(SIMPLE): Can't contact LDAP server (-1)` | `slapd` parado, URL incorrecta o puerto no accesible. | `sudo systemctl status slapd`, `sudo systemctl restart slapd`, `ss -tlnp \| grep 389`. Comprobar el `-H`. |
| 6 | `ping ldapserver.planetafp.local` falla / `Name or service not known` | Falta la línea en `/etc/hosts` o está mal escrita; el FQDN no coincide. | Revisar `/etc/hosts` (`20.0.0.5 ldapserver.planetafp.local ldapserver`) y comprobar con `getent hosts ldapserver.planetafp.local`. Recordar que cada cliente necesita su propia entrada. |
| 7 | `apt update` falla (`Temporary failure resolving`) | Netplan sin puerta de enlace o sin DNS. | Revisar `routes` y `nameservers` en el YAML, aplicar con `sudo netplan apply` y probar `ping 8.8.8.8`. |
| 8 | `netplan apply` muestra errores de YAML | Indentación incorrecta o tabuladores. | Usar solo espacios, validar con `sudo netplan generate` y corregir la línea indicada. |

---

*Playbook elaborado como guía operativa de despliegue de OpenLDAP. Sustituir cada marcador `[CAPTURA]` por la evidencia real de la práctica.*
