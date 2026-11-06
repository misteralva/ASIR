# Guía Básica de Virtualización y Proxmox VE

## 1. ¿Qué es un cluster de Proxmox?

Un **cluster de Proxmox** es la unión de dos o más servidores físicos que trabajan juntos como un solo sistema centralizado. 

Gracias a esto, puedes gestionar todas las máquinas virtuales y contenedores de varios servidores físicos desde una única pantalla o panel de control.

```
+-------------------------------------------------------+
|                PANEL DE CONTROL ÚNICO                 |
+-------------------------------------------------------+
                           |
        +------------------+------------------+
        |                                     |
+---------------+                     +---------------+
|  Servidor 1   | <--- Red de  --->   |  Servidor 2   |
| (Nodo Proxmox)|      Cluster        | (Nodo Proxmox)|
+---------------+                     +---------------+
```

---

## 2. ¿De qué otro modo se puede emular un cluster de Proxmox?

Si no dispones de varios servidores físicos reales para practicar, puedes **emular un cluster usando virtualización anidada**.

Esto significa instalar un programa de virtualización en tu ordenador personal (como VirtualBox o VMware Workstation) y, dentro de él, crear varias máquinas virtuales donde instalarás Proxmox VE. Luego, unes esas máquinas virtuales entre sí para formar el cluster.

---

## 3. ¿Qué tipo de tecnologías nos permite crear un cluster de alta disponibilidad?

La **Alta Disponibilidad (HA - High Availability)** garantiza que, si un servidor físico falla, las máquinas virtuales que estaban en él se reinicien automáticamente en otro servidor del cluster sin perder el servicio.

Para lograr esto se combinan tres tecnologías principales:
* **Corosync:** Es el sistema de comunicación que comprueba constantemente si todos los servidores del cluster están encendidos y funcionando ("latido" o *heartbeat*).
* **Almacenamiento compartido:** Permite que todos los servidores accedan a los mismos discos duros (ej. Ceph, NFS, iSCSI).
* **Gestor de HA (Proxmox HA Manager):** Decide qué hacer cuando un nodo falla y traslada el trabajo al nodo superviviente.

---

## 4. ¿Qué son los hipervisores?

Un **hipervisor** (o Monitor de Máquinas Virtuales) es un software especial que permite dividir los recursos de un ordenador real (CPU, RAM, disco) para crear y ejecutar varias máquinas virtuales al mismo tiempo de forma aislada.

```
+-------------------------------------------------------+
|  Máquina Virtual A   |  Máquina Virtual B             |
+-------------------------------------------------------+
|                     HIPERVISOR                        |
+-------------------------------------------------------+
|                 HARDWARE (CPU, RAM)                   |
+-------------------------------------------------------+
```

---

## 5. ¿Qué significa que sean de Tipo 1 o Tipo 2? Ejemplos

| Tipo | Descripción | Ejemplo |
| :--- | :--- | :--- |
| **Tipo 1 (Bare-Metal / Nativo)** | Se instala directamente sobre el hardware del equipo sin necesidad de un sistema operativo previo. Es muy rápido y eficiente. | **Proxmox VE**, VMware ESXi, Microsoft Hyper-V. |
| **Tipo 2 (Hosted / Alojado)** | Se instala como una aplicación más dentro de un sistema operativo tradicional (Windows, Linux, macOS). | **Oracle VirtualBox**, VMware Workstation. |

---

## 6. ¿En qué sistema operativo se basa Proxmox?

Proxmox VE está basado en **Debian GNU/Linux**. Utiliza un kernel Linux modificado para ofrecer un alto rendimiento y estabilidad empresarial.

---

## 7. ¿Qué tipos de virtualización están disponibles por defecto en Proxmox?

Proxmox ofrece dos tecnologías complementarias:

1. **KVM (Virtualización Completa):** Crea máquinas virtuales independientes con su propio sistema operativo (Windows, Linux, etc.). Ofrece un aislamiento total.
2. **LXC (Contenedores Linux):** Virtualización a nivel de sistema operativo. Comparte el kernel de Linux con el servidor principal, por lo que es extraordinariamente ligero y consume muy pocos recursos.

---

## 8. Términos clave y sus características

### **QEMU**
Es un emulador de hardware libre. En Proxmox trabaja junto con KVM para emular componentes físicos como tarjetas de red, discos gráficos y procesadores dentro de la máquina virtual.

### **KVM (Kernel-based Virtual Machine)**
Es una tecnología integrada en el kernel de Linux que transforma a Linux en un hipervisor de Tipo 1. Permite asignar CPU y memoria RAM reales a las máquinas virtuales con máximo rendimiento.

### **noVNC**
Es un cliente VNC escrito en HTML5 y JavaScript. Permite acceder a la pantalla o consola gráfica de las máquinas virtuales y contenedores directamente desde el navegador web, sin instalar programas adicionales.

### **CEPH**
Es un sistema de almacenamiento distribuido y definido por software. Permite agrupar los discos duros de los diferentes servidores del cluster para crear un único disco gigante, compartido y redundante. Si un disco o servidor se rompe, los datos no se pierden.

---

## 9. ¿Qué tipo de roles de usuarios y grupos podemos configurar en Proxmox?

Proxmox permite gestionar accesos mediante un sistema de **Control de Acceso Basado en Roles (RBAC)**:

* **Usuarios:** Cuentas individuales para personas o servicios (ej. `admin@pve`, `juan@pve`).
* **Grupos:** Conjuntos de usuarios para aplicar permisos de forma masiva (ej. `Grupo-Administradores`, `Grupo-Tecnicos`).
* **Roles (Permisos predefinidos):**
  * `Administrator`: Acceso total al sistema.
  * `NoAccess`: Sin acceso.
  * `PVEAdmin`: Administración de recursos sin modificar la configuración global del cluster.
  * `PVEVMUser`: Permite encender, apagar y consultar máquinas virtuales.
  * `PVEAuditor`: Modo lectura (solo ver estadísticas y configuraciones).

---

## 10. ¿Dónde se almacenan las ISOs y templates en Proxmox?

Las imágenes ISO para instalar sistemas y las plantillas (*templates*) de contenedores se guardan en los **almacenamientos de archivos** configurados en Proxmox.

Por defecto, se guardan en el almacenamiento local llamado **`local`** dentro de las siguientes rutas del servidor:

```
Ruta base principal:
/var/lib/vz/

Subcarpetas específicas:
├── /var/lib/vz/template/iso/       <-- (Imágenes ISO para VM)
└── /var/lib/vz/template/cache/     <-- (Plantillas para contenedores LXC)
```

> **Nota:** Si utilizas almacenamiento de red (como NFS o CIFS), las ISOs se guardarán en la carpeta compartida correspondiente que hayas asignado desde el panel web.
