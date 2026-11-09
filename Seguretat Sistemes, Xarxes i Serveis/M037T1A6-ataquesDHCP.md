# 🛡️ Guía Práctica: Simulación de DHCP Starvation y Mitigación mediante Rate Limiting en iptables & Kea DHCP

**Módulo:** 037 - Administración de Sistemas Operativos / Redes  
**Entorno:** Laboratorio de pruebas en VirtualBox (Entorno Aislado)  
**Autor:** David  
**Fecha:** 2026  

---

## 📋 Descripción del Proyecto

Este laboratorio documenta la simulación de un ataque de **Agotamiento de Direcciones IP (DHCP Starvation)** desde una máquina de auditoría (**Kali Linux**) mediante la herramienta **Yersinia**, y el posterior despliegue de medidas de protección (**Hardening**) en el Servidor DHCP (**Ubuntu Server** con **Kea DHCP**) utilizando el módulo `hashlimit` de **iptables** y la monitorización de tráfico a bajo nivel con **tcpdump**.

---

## 📐 Topología de Red y Componentes

```
                   +------------------------------------+
                   |        VirtualBox Hypervisor       |
                   +------------------------------------+
                                     |
               +---------------------+---------------------+
               |                                           |
       [ Interfaz NAT ]                            [ Interfaz NAT ]
               |                                           |
+------------------------------+           +------------------------------+
|     VM Servidor (Ubuntu)     |           |     VM Atacante (Kali)       |
|  - enp0s3: 10.0.2.15 (NAT)   |           |  - eth0: 10.0.2.x (NAT)      |
|  - enp0s8: 10.10.10.1/24     |           |  - eth1: 10.10.10.5/24 (sec) |
|  - Servicio: Kea DHCP        |           |  - Herramientas: Yersinia,    |
|  - Firewall: iptables        |           |    tcpdump                   |
+--------------+---------------+           +--------------+---------------+
               |                                           |
               +---------------[ RED INTERNA ]-------------+
                         (Modo Promiscuo: Allow All)
```

### Tabla de Direccionamiento IP

| Máquina | Interfaz | Tipo de Red | Dirección IP / Máscara | Función |
| :--- | :--- | :--- | :--- | :--- |
| **Ubuntu Server** | `enp0s3` | NAT | `10.0.2.15/24` | Salida a Internet / Gestión |
| **Ubuntu Server** | `enp0s8` | Red Interna | `10.10.10.1/24` | Gateway / Servidor Kea DHCP |
| **Kali Linux** | `eth0` | NAT | Dinámica (`10.0.2.x`) | Descargas / Gestión |
| **Kali Linux** | `eth1` | Red Interna | `10.10.10.2` (dinámica) <br> `10.10.10.5/24` (estática) | Inyección de paquetes / Auditoría |

> **Requisito Crítico de VirtualBox:** En las interfaces de Red Interna (`enp0s8` y `eth1`), el **Modo Promiscuo** debe configurarse obligatoriamente en **"Permitir todo" (Allow All)** en los ajustes del hipervisor.

---

## 🚀 Paso 1: Configuración del Enrutamiento (NAT) e IPTables Básico

Antes de iniciar las pruebas de seguridad, configuramos el Servidor Ubuntu para actuar como router enmascarando el tráfico de la red interna hacia la interfaz externa.

### 1.1 Habilitar IP Forwarding en el Kernel
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```
*(Para hacer persistente el cambio tras reiniciar, edita `/etc/sysctl.conf` y añade `net.ipv4.ip_forward=1`).*

### 1.2 Configurar el Enmascaramiento (MASQUERADE) y Forwading
```bash
# Enmascarar tráfico saliente a Internet por la interfaz NAT
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

# Permitir el reenvío entre la red interna y la interfaz externa
sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

---

## ⚡ Paso 2: Ejecución del Ataque DHCP Starvation (Fase Vulnerable)

Demostramos el agotamiento del pool de direcciones del servidor Kea DHCP sin reglas de protección activas.

### 2.1 Configuración de la interfaz en Kali Linux
Asignamos la IP secundaria en la interfaz interna de Kali (`eth1`):
```bash
sudo ip addr add 10.10.10.5/24 dev eth1
sudo ip link set eth1 up
```

### 2.2 Lanzar Yersinia desde Kali Linux
Ejecutamos Yersinia en modo interactivo:
```bash
sudo yersinia -I
```
1. Presiona `i` para abrir el menú de interfaces y selecciona **`eth1`**.
2. Presiona `g` y selecciona el protocolo **DHCP**.
3. Presiona `x` y ejecuta la **Opción 1** (*sending RAW / DISCOVER packets*).
4. Transcurridos 15 a 20 segundos, presiona `L` (mayúscula) para detener el ataque y `q` para salir.

### 2.3 Verificación del Agotamiento en Kea DHCP
En el **Servidor Ubuntu**, comprobamos las concesiones otorgadas por Kea DHCP en su archivo de base de datos CSV (`/var/lib/kea/kea-leases4.csv`):
```bash
cat /var/lib/kea/kea-leases4.csv | wc -l
```
* **Resultado:** El archivo se llena masivamente de registros e IPs asignadas a direcciones MAC aleatorias generadas por Yersinia, agotando el pool disponible.

---

## 🛡️ Paso 3: Configuración de la Mitigación (Rate Limiting con iptables)

Implementamos un control de tasa por limitación de frecuencia (*hashlimit*) en la cadena `INPUT` de `iptables` sobre la interfaz `enp0s8` para frenar la inundación de peticiones `DHCPDISCOVER`.

### 3.1 Limpieza del Pool de Concesiones en Kea DHCP
Reseteamos el archivo de concesiones para iniciar la prueba de defensa desde cero:
```bash
sudo systemctl stop kea-dhcp4-server
sudo echo "address,hwaddr,client_id,valid_lifetime,expire,subnet_id,fqdn_fwd,fqdn_rev,hostname,state,user_context,pool_id" | sudo tee /var/lib/kea/kea-leases4.csv
sudo systemctl start kea-dhcp4-server
```

### 3.2 Aplicación del Filtro Anti-Starvation con iptables
```bash
sudo iptables -A INPUT -i enp0s8 -p udp --dport 67 \
 -m hashlimit --hashlimit-name dhcp_limit \
 --hashlimit-above 3/sec --hashlimit-burst 5 \
 --hashlimit-htable-expire 10000 \
 -j DROP
```

---

## 📊 Paso 4: Comprobación y Métodos de Verificación de la Defensa

Lanzamos nuevamente el ataque de Yersinia desde Kali Linux por `eth1`.

### 4.1 Método A: Lectura de Contadores de iptables
Monitoreamos en el servidor la cantidad de paquetes filtrados en tiempo real:
```bash
sudo iptables -L INPUT -v -n
```

**Resultado obtenido:**
```text
Chain INPUT (policy ACCEPT 15559 packets, 11M bytes)
 pkts bytes target     prot opt in     out     source               destination
 138K   38M DROP       17   --  enp0s8 *       0.0.0.0/0            0.0.0.0/0            udp dpt:67 limit: above 3/sec burst 5 htable-expire 10000
```
> **Evidencia:** El cortafuegos descartó exitosamente **138,000 paquetes** (38 MB de tráfico malicioso), bloqueando la inundación antes de que afectara al demonio DHCP.

### 4.2 Método B: Verificación de Integridad en Kea DHCP
```bash
cat /var/lib/kea/kea-leases4.csv | wc -l
```
* **Resultado obtenido:** `2` (únicamente la línea de cabecera CSV y la concesión previa). El pool de direcciones se mantuvo limpio y protegido.

### 4.3 Método C: Escucha de Tráfico a Bajo Nivel con tcpdump
En el servidor ejecutamos la escucha de paquetes UDP al puerto 67:
```bash
sudo tcpdump -i enp0s8 -n udp port 67
```
* **Análisis de la captura:** `tcpdump` registra las tramas `DHCPDISCOVER` entrantes en el adaptador de red, pero debido al descarte de `iptables`, el servidor Kea jamás emite respuestas `DHCPOFFER`.

---

## 📝 Paso 5: Cuestionario Teórico y Análisis Avanzado

### 1. Análisis de Capas
* **¿Por qué alterar simultáneamente la MAC (L2) y el campo `chaddr` (L7)?**  
  El servidor DHCP consulta el campo `chaddr` (*Client Hardware Address*) del protocolo BOOTP/DHCP (Capa 7) para asignar direcciones IP. Sin embargo, las tarjetas de red y los dispositivos de conmutación intermedios procesan la dirección MAC de origen de la trama Ethernet (Capa 2). Para ejecutar un ataque efectivo que esquive filtros simples y obligue a asignar nuevas IPs, es imprescindible falsificar ambos campos en sincronía.
* **¿Qué ocurre si solo se modifica uno de los campos?**  
  * *Solo `chaddr` (L7):* Las reglas de firewall o switches detectan miles de solicitudes originadas por una misma dirección MAC física (L2), activando defensas por tasa de envío de la trama.
  * *Solo MAC (L2):* El servidor DHCP lee el mismo valor en `chaddr` (L7), interpretando que se trata del mismo equipo intentando renovar su dirección IP, por lo que responde con la misma IP y no agota el pool.

### 2. Impacto en Entornos Virtuales
* **¿Qué habría ocurrido sin activar el "Modo Promiscuo" en VirtualBox?**  
  El hipervisor habría bloqueado todas las tramas maliciosas generadas por Yersinia. Por defecto, VirtualBox descarta cualquier trama saliente cuyo campo MAC de origen no coincida exactamente con la dirección MAC asignada a la interfaz virtual de la VM. El modo promiscuo en **"Permitir todo" (Allow All)** es obligatorio para simular el comportamiento de una red física real donde las tramas falsificadas circulan libremente por el medio.

### 3. Optimización con Linux Bridge (`ebtables`)
En arquitecturas donde la interfaz actúa como puente en capa de enlace (`br0`), es posible validar que la MAC de la trama Ethernet (L2) coincida con la dirección `chaddr` del paquete DHCP (L7) mediante `ebtables`:
```bash
sudo ebtables -A FORWARD -p IPv4 --ip-proto udp --ip-dport 67 \
  --dhcp-status discover \
  ! --dhcp-mac-match \
  -j DROP
```

### 4. Resistencia de la Mitigación y Alternativas en Switches
* **Limitación del Rate Limiting en Linux:**  
  Si un atacante genera un volumen masivo de direcciones MAC distintas en Capa 2 a muy alta velocidad, la tabla de estado hash del módulo `hashlimit` del kernel de Linux puede llegar a sufrir agotamiento de memoria RAM.
* **Soluciones Definitivas en Switches de Acceso (Enterprise):**  
  1. **DHCP Snooping:** Divide las interfaces del switch en *Trusted* (conectadas a servidores DHCP autorizados) y *Untrusted* (conectadas a usuarios/clientes). En las interfaces *Untrusted* limita la tasa máxima de paquetes DHCP permitidos por segundo (`ip dhcp snooping limit rate 10`).
  2. **Port Security:** Restringe la cantidad máxima de direcciones MAC aprendidas en un puerto físico del switch. Si un puerto supera el límite establecido (ej. máximo 2 MACs), el switch conmuta el puerto a estado de bloqueo automático (*err-disable*).

---

## 📌 Guardar Reglas de Forma Permanente

Para guardar las reglas de `iptables` de manera permanente en Ubuntu Server:
```bash
sudo apt update && sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```
