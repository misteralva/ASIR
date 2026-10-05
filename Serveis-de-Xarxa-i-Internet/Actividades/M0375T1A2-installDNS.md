# Guía completa desde cero: servidor DNS primario (BIND9) en Debian 13 (Trixie)

Este documento recoge, paso a paso, todo el proceso: comprobación de los parámetros de red y dominio del sistema, instalación de BIND9, configuración de las zonas directa e inversa, aseguramiento del servicio y verificación final.

## Índice

1. [Ficha técnica y verificación de datos del sistema](#1-ficha-técnica-y-verificación-de-datos-del-sistema)
2. [Configuración de la red estática en Debian](#2-configuración-de-la-red-estática-en-debian)
3. [Instalación del servidor DNS (BIND9)](#3-instalación-del-servidor-dns-bind9)
4. [Declaración de zonas (named.conf.local)](#4-declaración-de-zonas-namedconflocal)
5. [Creación de ficheros de zona (directa e inversa)](#5-creación-de-ficheros-de-zona-directa-e-inversa)
6. [Configuración de opciones globales (named.conf.options)](#6-configuración-de-opciones-globales-namedconfoptions)
7. [Forzar el uso de IPv4 (/etc/default/named)](#7-forzar-el-uso-de-ipv4-etcdefaultnamed)
8. [Puesta en marcha y fijado del resolvedor (/etc/resolv.conf)](#8-puesta-en-marcha-y-fijado-del-resolvedor-etcresolvconf)
9. [Pruebas de verificación con nslookup](#9-pruebas-de-verificación-con-nslookup)

---

## 1. Ficha técnica y verificación de datos del sistema

Antes de empezar, verificamos las características de la máquina virtual y los datos del dominio asignado.

| Parámetro | Configuración / Valor |
| :--- | :--- |
| **Sistema operativo** | Debian GNU/Linux 13 (Trixie) |
| **Tipo de red (VirtualBox)** | Red NAT |
| **Disco duro (HDD)** | 25 GB |
| **Memoria RAM** | 2 GB |
| **Dirección IP estática** | `192.168.6.100` |
| **Máscara de red** | `255.255.255.0` (`/24`) |
| **Puerta de enlace (router)** | `192.168.6.1` |
| **Nombre del equipo (hostname)** | `debian` |
| **Dominio del sistema** | `myguest.virtualbox.org` |
| **FQDN del servidor** | `debian.myguest.virtualbox.org` |
| **Servidor de nombres (NS)** | `ns.myguest.virtualbox.org` |

### Comprobación del hostname y del dominio

Podemos ver el nombre de host y el dominio asignados ejecutando:

```bash
# Ver el hostname del equipo
hostname

# Ver el FQDN (nombre de dominio completamente cualificado)
hostname -f
```

Si es necesario ajustar el nombre del sistema para que coincida con el dominio:

```bash
sudo hostnamectl set-hostname debian
```

---

## 2. Configuración de la red en Debian

Para que el servidor DNS funcione de forma fiable, su dirección IP debe ser fija. Así que el adaptador de red 1 pondremos una red NAT con DHCP y en el segundo adaptador de Red Interna pondremos una IP estática

### Paso 2.1: identificar la interfaz de red

Comprobamos el nombre de la tarjeta de red activa:

```bash
ip a
```

### Paso 2.2: editar `/etc/network/interfaces`

Abrimos el archivo de configuración de red:

```bash
sudo nano /etc/network/interfaces
```

Establecemos la configuración IP estática:

```text
# Interfaz de bucle local (loopback)
auto lo
iface lo inet loopback

# Interfaz primária de red dinámica
allow-hotplug enp0s3
iface enp0s3 inet dhcp

# Interfaz secundária de red estática
allow-hotplug enp0s8
iface enp0s8 inet static
    address 192.168.6.100
    netmask 255.255.255.0
    broadcast 192.168.6.225
    dns-nameservers 192.168.6.100
    dns-search myguest.virtualbox.org
```

### Paso 2.3: reiniciar la red y comprobar la IP

Aplicamos los cambios en el servicio de red:

```bash
sudo systemctl restart networking
```

Verificamos que la interfaz tiene asignada la IP `10.0.2.15` para enp0s3 y `192.168.6.100` para enp0s8:

```bash
ip a show enp0s3
ip a show enp0s8
```

---

## 3. Instalación del servidor DNS (BIND9)

Actualizamos el índice de paquetes e instalamos BIND9 y sus utilidades:

```bash
sudo apt update
sudo apt install bind9 bind9-utils -y
```

---

## 4. Declaración de zonas (`named.conf.local`)

### Paso 4.1: crear la carpeta para las zonas

Creamos la carpeta `/etc/bind/zones` para almacenar los ficheros de zona:

```bash
sudo mkdir -p /etc/bind/zones
```

### Paso 4.2: configurar `named.conf.local`

Editamos el archivo de declaración de zonas:

```bash
sudo nano /etc/bind/named.conf.local
```

Añadimos las zonas directa e inversa:

```text
// Zona directa: asigna nombres de dominio a direcciones IP
zone "myguest.virtualbox.org" {
    type master;
    file "/etc/bind/zones/db.myguest.virtualbox.org";
};

// Zona inversa: asigna direcciones IP a nombres de dominio
zone "6.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.6.168.192";
};
```

Verificamos la sintaxis del archivo:

```bash
sudo named-checkconf
```

---

## 5. Creación de ficheros de zona (directa e inversa)

### Paso 5.1: fichero de zona directa

Ruta: `/etc/bind/zones/db.myguest.virtualbox.org`

Creamos el fichero de resolución directa:

```bash
sudo nano /etc/bind/zones/db.myguest.virtualbox.org
```

Añadimos los registros DNS:

```text
; BIND data fila
;
$TTL    604800
@       IN      SOA     myguest.virtualbox.org. debian.myguest.virtualbox.org. (
                              2         ; se = Serial 
                            12h         ; ref = Refresh
                            15m         ; ref = Retry
                             3w         ; ex = Expire
                             2h        ; nx = nxdomain TTL
                              )

; name   servers  - NS records
@       IN      NS      ns.myguest.virtualbox.org.

; name   servers  - A records
ns      IN      A       192.168.6.100

; hosts - A records
@       IN      A       192.168.6.100
debian  IN      A       192.168.6.100

```

Verificamos la sintaxis de la zona directa:

```bash
sudo named-checkzone myguest.virtualbox.org /etc/bind/zones/db.myguest.virtualbox.org
```

### Paso 5.2: fichero de zona inversa

Ruta: `/etc/bind/zones/db.6.168.192`

Creamos el fichero de resolución inversa:

```bash
sudo nano /etc/bind/zones/db.6.168.192
```

Añadimos los registros PTR:

```text
; BIND data fila
;
$TTL    604800
@       IN      SOA     myguest.virtualbox.org. debian.myguest.virtualbox.org. (
                              2         ; se = Serial
                            12h         ; ref = Refresh
                            15m         ; ref = Retry
                             3w         ; ex = Expire
                             2h         ; nx = nxdomain TTL
                              )



; name   servers  - A records
@         IN      NS        ns.

; PTR Records
100       IN      PTR       ns.myguest.virtualbox.org.
```

Verificamos la sintaxis de la zona inversa:

```bash
sudo named-checkzone 6.168.192.in-addr.arpa /etc/bind/zones/db.6.168.192
```

---

## 6. Configuración de opciones globales (`named.conf.options`)

Configuramos permisos, ACL, interfaces de escucha, recursión y reenviadores DNS.

Abrimos el archivo:

```bash
sudo nano /etc/bind/named.conf.options
```

Escribimos la siguiente configuración:

```text
acl "safeclients" {
    localhost;
    192.168.6.100;
    localnets;
};

options {
    directory "/var/cache/bind";
    recursion yes;
    allow-recursion { safeclients; };
    listen-on { any; };
    allow-transfer { none; };
    allow-query { safeclients; };
    allow-query-cache { safeclients; };
    
    forwarders {
        1.1.1.1;
        9.9.9.9;
    };
};
```

Verificamos la sintaxis global:

```bash
sudo named-checkconf
```

---

## 7. Forzar el uso de IPv4 (`/etc/default/named`)

Para evitar retrasos por búsquedas IPv6 inalcanzables, forzamos el parámetro `-4`.

Abrimos el archivo:

```bash
sudo nano /etc/default/named
```

Ajustamos la variable:

```text
OPTIONS="-u bind -4"
```

---

## 8. Puesta en marcha y fijado del resolvedor (`/etc/resolv.conf`)

### Paso 8.1: reinicio y estado de BIND9

Reiniciamos el servicio para cargar toda la configuración:

```bash
sudo systemctl restart bind9
```

Comprobamos que esté activo:

```bash
sudo systemctl status bind9
```

### Paso 8.2: ajuste y bloqueo de `/etc/resolv.conf`

Ajustamos el resolvedor local del sistema:

```bash
sudo nano /etc/resolv.conf
```

Añadimos:

```text
search myguest.virtualbox.org
nameserver 192.168.6.100
```

Bloqueamos el archivo contra modificaciones automáticas del sistema o de la red:

```bash
sudo chattr +i /etc/resolv.conf
```

> Si más adelante necesitas volver a editarlo, quita antes el bloqueo con `sudo chattr -i /etc/resolv.conf`.

---

## 9. Pruebas de verificación con `nslookup`

Realizamos las comprobaciones finales desde la terminal.

**1. Resolución directa del servidor de nombres:**

```bash
nslookup ns.myguest.virtualbox.org
```

**2. Resolución directa del host del sistema:**

```bash
nslookup debian.myguest.virtualbox.org
```

**3. Resolución por el dominio raíz:**

```bash
nslookup myguest.virtualbox.org
```

**4. Resolución inversa:**

```bash
nslookup 192.168.6.100
```

**5. Resolución de dominios externos (forwarders):**

```bash
nslookup google.com
```
