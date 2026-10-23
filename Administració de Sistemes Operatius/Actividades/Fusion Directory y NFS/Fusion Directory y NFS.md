# PLAYBOOK PROCEDIMENTAL: INTEGRACIÓN DE FUSION DIRECTORY Y ALMACENAMIENTO NFS EN PROXMOX / HONEYNET

**Documento:** Manual de Operaciones / Guía de Despliegue  
**Proyecto:** HoneyNet (ASIX - Perfil Ciberseguridad)  
**Autores:** Álex Naranjo Blázquez y David Álvarez Fernández  
**Dominio Base LDAP:** `dc=honeynet,dc=local`  
**Recurso Compartido:** `/srv/nfs/telemetria`  

---

## 1. Prerrequisitos de la Infraestructura

Antes de iniciar la integración del panel web y el almacenamiento compartido en el entorno virtualizado de Proxmox, se verifica la existencia y operatividad de la infraestructura base:

1. **Servidor OpenLDAP (`slapd`):** En ejecución activa dentro de la red interna de la HoneyNet con el Base DN `dc=honeynet,dc=local` y la cuenta de administración activa (`cn=admin,dc=honeynet,dc=local`).
2. **Integración NSS (`nslcd` / `sssd`):** Instalada y activa tanto en el servidor central como en los nodos de análisis para resolver identidades LDAP localmente mediante el sistema operativo (`getent passwd`, `getent group`).
3. **Estructura base:** Existencia del grupo POSIX de analistas de seguridad (`analistas`) e identidades de prueba registradas en el directorio LDAP.

---

## 2. Módulo A: Gestión e Instalación de Fusion Directory

### 2.1. Instalación de Dependencias y Repositorio Oficial
En la máquina virtual/LXC que actúa como servidor de identidad web, se actualizan los repositorios e instalan el servidor web Apache, PHP con sus extensiones requeridas y la herramienta de conversión de esquemas:

```bash
# Actualización e instalación del stack Apache + PHP
sudo apt update
sudo apt install -y apache2 php php-ldap php-mbstring php-xml php-curl php-imap schema2ldif
```

A continuación, se añade la clave GPG y el repositorio oficial de Fusion Directory:

```bash
# Importación de la clave pública oficial del repositorio
wget -qO - https://repos.fusiondirectory.org/fusiondirectory.gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/fusiondirectory.gpg

# Adición del repositorio oficial (distribución Debian/Ubuntu)
echo "deb http://repos.fusiondirectory.org/fusiondirectory-1.3/debian/bookworm bookworm main" | sudo tee /etc/apt/sources.list.d/fusiondirectory.list

# Actualización e instalación del Core de Fusion Directory
sudo apt update
sudo apt install -y fusiondirectory fusiondirectory-schema
```

> ⚠️ **Advertencia técnica:** Se evita explícitamente la instalación del paquete `fusiondirectory-plugins-all` para prevenir la sobrecarga de dependencias innecesarias que puedan comprometer la seguridad o estabilidad de la HoneyNet.

---

### 2.2. Gestión e Importación de Esquemas LDAP
Para que OpenLDAP reconozca la estructura de datos de Fusion Directory, se inyectan los esquemas nativos en tiempo de ejecución (`cn=config`):

```bash
# Inyección de los esquemas base
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/core-fd.schema
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/core-fd-conf.schema
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/ldapns.schema
sudo fusiondirectory-schema-manager --insert-schema /etc/ldap/schema/fusiondirectory/template-fd.schema
```

---

### 2.3. Ejecución del Asistente Web de Configuración (`/fusiondirectory/setup`)
Se accede desde el navegador web a la dirección: `http://<IP_SERVIDOR_HONEYNET>/fusiondirectory/`.

1. **Paso 1 (Welcome / Token):** Se valida el acceso creando el token físico en la consola del servidor:
   ```bash
   echo "token_seguro_honeynet" | sudo tee /var/spool/fusiondirectory/setup_token
   ```
2. **Paso 2 (Language):** Selección del idioma de administración (Español / Inglés).
3. **Paso 3 (Checks):** Verificación de compatibilidad (validación en verde de extensiones PHP como `php-ldap`, `mbstring`, etc.).
4. **Paso 4 (LDAP Connection):** Configuración de la conexión al directorio:
   * **Base DN:** `dc=honeynet,dc=local`
   * **Bind DN:** `cn=admin,dc=honeynet,dc=local`
   * **Password:** Credenciales del administrador de OpenLDAP.
5. **Paso 5 (Configuration):** Definición de las Unidades Organizativas (`ou=usuarios`, `ou=grupos`).
6. **Paso 6 (Inspection & Finalization):** Guardado del fichero de configuración en `/etc/fusiondirectory/fusiondirectory.conf`.

---

### 2.4. Administración Visual de Usuarios y Grupos POSIX
Una vez completado el asistente, se gestionan las identidades desde el panel web:

1. **Alta del Grupo POSIX:**
   * Navegar a **Gestión de Usuarios y Grupos** -> **Groups** -> **Add**.
   * Crear el grupo `analistas` asignándole un **GID POSIX** (ejemplo: `10000`).
2. **Alta del Usuario Analista (Perfil POSIX):**
   * Navegar a **Users** -> **Add**.
   * Rellenar datos personales (*Nombre*, *Apellidos*, *Email*), ir a la pestaña **POSIX** y asignar el UID, ruta home (`/home/anaranjo` o `/home/dalvarez`) y shell bash.
   * Asignar contraseña e integrar el usuario dentro del grupo `analistas`.

---

## 3. Módulo B: Despliegue e Integración del Servidor NFS

### 3.1. Instalación y Preparación del Recurso Compartido
El recurso NFS actuará como almacenamiento centralizado para los logs, capturas pcap y telemetría de ataques capturados por los sensores cebo de la HoneyNet:

```bash
# Instalación del servidor NFS
sudo apt install -y nfs-kernel-server

# Creación del directorio para el almacenamiento de telemetría
sudo mkdir -p /srv/nfs/telemetria
```

---

### 3.2. Permisos POSIX Vinculados al Grupo LDAP
Mapeo de la carpeta local con el grupo `analistas` de LDAP usando el servicio de nombres NSS:

```bash
# Propietario root y grupo LDAP analistas
sudo chown root:analistas /srv/nfs/telemetria

# Restricción estricta 770 (rwxrwx---)
sudo chmod 770 /srv/nfs/telemetria
```

---

### 3.3. Exportación de la Red en `/etc/exports`
Definición de la subred interna de la HoneyNet autorizada a montar el recurso:

```text
/srv/nfs/telemetria 192.168.1.0/24(rw,sync,no_root_squash)
```

Aplicación de la regla de exportación:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

---

### 3.4. Montaje y Persistencia en Clientes/Nodos
En las máquinas de análisis o nodos clientes de la red:

```bash
# Instalación del paquete cliente y punto de montaje
sudo apt install -y nfs-common
sudo mkdir -p /mnt/nfs/telemetria

# Prueba de montaje temporal
sudo mount 192.168.1.5:/srv/nfs/telemetria /mnt/nfs/telemetria
```

Inclusión en `/etc/fstab` para montaje automático al iniciar:

```text
192.168.1.5:/srv/nfs/telemetria /mnt/nfs/telemetria nfs defaults 0 0
```

```bash
sudo mount -a
```

---

## 4. Módulo C: Verificación y Troubleshooting

### 4.1. Comandos de Comprobación

* **En el Servidor NFS:**
  ```bash
  # Verificación de recursos exportados activamente
  sudo exportfs -v

  # Comprobar propietario y permisos de la carpeta
  ls -ld /srv/nfs/telemetria
  # Salida esperada: drwxrwx--- 2 root analistas ... /srv/nfs/telemetria
  ```

* **En el Cliente/Nodo:**
  ```bash
  # Listar recursos compartidos expuestos por el servidor
  showmount -e 192.168.1.5

  # Confirmar que el volumen NFS está correctamente montado
  df -h | grep nfs
  ```

---

### 4.2. Prueba Práctica de Segregación de Acceso

#### Escenario 1: Usuario No Autorizado (Usuario Local)
El usuario `ubuntu` o cualquier cuenta local no perteneciente al grupo `analistas` intenta explorar el directorio:

```bash
ubuntu@nodo-cliente:~$ ls /mnt/nfs/telemetria
ls: cannot open directory '/mnt/nfs/telemetria': Permission denied
```

* **Resultado:** Acceso bloqueado por el permiso `770`.

#### Escenario 2: Usuario Autorizado (Analista LDAP)
Un analista (`anaranjo` o `dalvarez`), autenticado mediante LDAP y perteneciente al grupo `analistas`, accede al recurso:

```bash
anaranjo@nodo-cliente:~$ cd /mnt/nfs/telemetria
anaranjo@nodo-cliente:/mnt/nfs/telemetria$ touch log_ataque_cowrie.json
anaranjo@nodo-cliente:/mnt/nfs/telemetria$ ls -l
-rw-r--r-- 1 anaranjo analistas 0 Oct 23 17:35 log_ataque_cowrie.json
```

* **Resultado:** Acceso y escritura concedidos correctamente gracias a la resolución de identidades LDAP mediante NSS.
