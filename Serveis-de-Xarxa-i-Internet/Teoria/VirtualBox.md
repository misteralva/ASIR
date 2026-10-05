# Guía de redes en VirtualBox

Guía práctica para entender y configurar los modos de red de VirtualBox: qué hace cada uno, cuándo usarlo y cómo configurarlo desde la interfaz gráfica y desde la línea de comandos.

> Basada y reexplicada a partir del artículo original de NAKIVO: [Configuración de redes de VirtualBox](https://www.nakivo.com/es/blog/virtualbox-network-setting-guide/). Las capturas pertenecen al artículo original.

## Índice

1. [Adaptadores de red virtuales](#1-adaptadores-de-red-virtuales)
2. [Modelos de tarjeta de red (NIC)](#2-modelos-de-tarjeta-de-red-nic)
3. [Modos de red](#3-modos-de-red)
4. [Tabla comparativa](#4-tabla-comparativa)
5. [Reenvío de puertos](#5-reenvío-de-puertos-port-forwarding)
6. [Chuleta de comandos VBoxManage](#6-chuleta-de-comandos-vboxmanage)
7. [Conclusión](#7-conclusión)

---

## 1. Adaptadores de red virtuales

Cada máquina virtual (VM) puede tener **hasta 8 tarjetas de red virtuales** (NIC):

- En la **interfaz gráfica** solo se pueden configurar **4** (pestañas *Adapter 1* a *Adapter 4*).
- Con la herramienta de línea de comandos **`VBoxManage`** se pueden configurar las **8**.

Para llegar a la configuración: selecciona la VM → **Settings** → **Network**.

![VirtualBox Network Settings for Adapter 1](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-Network-Settings-for-Adapter-1.webp)

Puntos clave de esta pantalla:

- **Enable Network Adapter**: activa o desactiva la tarjeta. Es como conectar o quitar la NIC física del equipo. Por defecto, el Adaptador 1 viene activado.
- **Attached to**: aquí eliges el **modo de red** (lo veremos en detalle más abajo).
- **Advanced**: despliega opciones extra como el modelo de tarjeta, la dirección MAC, el modo promiscuo y la casilla *Cable Connected*.

---

## 2. Modelos de tarjeta de red (NIC)

VirtualBox no usa una tarjeta real, sino que **emula por software** un modelo concreto. Puedes elegir entre seis:

| Modelo | Cuándo usarlo |
|---|---|
| **AMD PCnet-PCI II (Am79C970A)** | Sistemas muy antiguos (por ejemplo Windows 2000). Windows 7, 8 y 10 no traen driver para ella. Chip de 10 Mbit. |
| **AMD PCnet-FAST III (Am79C973)** | Compatible con casi cualquier sistema invitado. GRUB puede arrancar por red con ella (PXE). |
| **Intel PRO/1000 MT Desktop (82540EM)** | Recomendada para Windows Vista y posteriores, y para la mayoría de distribuciones Linux. |
| **Intel PRO/1000 T Server (82543GC)** | Windows XP la reconoce sin instalar drivers. |
| **Intel PRO/1000 MT Server (82545EM)** | Útil al importar plantillas OVF creadas en otras plataformas. |
| **Paravirtualized Network (virtio-net)** | Caso especial: en vez de emular hardware, el sistema invitado usa una interfaz de software pensada para virtualización. Menos sobrecarga, mejor rendimiento. Requiere Linux con kernel 2.6.25 o superior, o drivers VirtIO en Windows. |

### Jumbo frames

Las *jumbo frames* son tramas Ethernet con más de 1500 bytes de carga. VirtualBox las soporta de forma limitada:

- Necesitas una tarjeta **Intel** y el modo **Bridged**.
- Con tarjetas **AMD** se descartan en silencio, tanto de entrada como de salida.
- Están **desactivadas por defecto**.

---

## 3. Modos de red

Cada adaptador se configura **de forma independiente**. Por ejemplo, el Adaptador 1 puede estar en NAT y el Adaptador 2 en Host-only a la vez.

![Seleccionar el modo de red](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-selecting-a-network-mode-for-the-virtual-network-adapter.webp)

Los nombres que usa `VBoxManage` para cada modo son:

```
none, null, nat, natnetwork, bridged, intnet, hostonly, generic
```

### 3.1 Not attached (no conectado)

La tarjeta existe, pero **no tiene conexión**, como si hubieras desenchufado el cable Ethernet.

**Para qué sirve:** pruebas. Por ejemplo, simular una caída de red para comprobar si un cliente DHCP recupera IP, o si una descarga se reanuda tras un corte.

> 💡 Puedes conseguir lo mismo con cualquier otro modo desmarcando **Cable Connected**, incluso con la VM encendida. Recuerda pulsar **OK** para aplicar el cambio.

### 3.2 NAT

Es el **modo por defecto**. La VM sale a Internet (y a la LAN) a través de un router NAT virtual que integra VirtualBox, usando la tarjeta de red del anfitrión como salida.

- ✅ La VM puede acceder a Internet y a otros equipos de la red.
- ❌ Nadie puede acceder a la VM desde fuera (ni el anfitrión ni otros equipos), salvo que configures **reenvío de puertos**.
- Cada VM en modo NAT está en **su propia red aislada**, así que dos VM en NAT **no se ven entre sí**.

**Direccionamiento por defecto:**

| Dato | Valor |
|---|---|
| IP de la VM (por DHCP) | `10.0.2.15` |
| Puerta de enlace | `10.0.2.2` |
| Máscara | `255.255.255.0` |

Estas direcciones no se pueden cambiar desde la interfaz gráfica. Como todas las VM en NAT reciben la misma IP, es normal que dos VM tengan `10.0.2.15` al mismo tiempo: cada una vive detrás de su propio NAT privado.

![Cómo funciona el modo NAT](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-modes-–-how-the-NAT-mode-works.webp)

**Ideal para:** una VM que solo necesita salir a Internet (actualizar paquetes, navegar, descargar).

**Comando:**

```bash
VBoxManage modifyvm "NombreVM" --nic1 nat
```

- `NombreVM`: nombre de tu máquina virtual.
- `nic1`: número del adaptador (de `nic1` a `nic8`).
- `nat`: modo de red.

Las reglas de reenvío de puertos de este modo se configuran **por VM**, en `VM > Settings > Network > Advanced > Port Forwarding`.

### 3.3 NAT Network (red NAT)

Funciona como el NAT normal, pero **varias VM comparten la misma red NAT** y por tanto **sí pueden comunicarse entre ellas**. Es el equivalente a tener varias máquinas detrás de un mismo router doméstico.

- ✅ Las VM se ven entre sí.
- ✅ Las VM salen a Internet y a la LAN.
- ❌ Desde fuera (LAN, Internet o el propio anfitrión) no se puede acceder a ellas, salvo con reenvío de puertos.

![Modo NAT Network](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-the-NAT-Network-mode.webp)

**Configuración global.** La red NAT se crea y edita en `File > Preferences > Network` (doble clic sobre la red existente, o `+` / `x` para añadir o borrar).

![Editar la red NAT en las preferencias globales](https://www.nakivo.com/blog/wp-content/uploads/2019/07/Global-VirtualBox-network-settings-–-editing-the-settings-of-the-NAT-Network.webp)

Desde la ventana emergente puedes activar o desactivar DHCP e IPv6 y configurar el reenvío de puertos.

![Configurar la red NAT](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-configuring-the-NAT-Network.webp)

**Direccionamiento:**

- Red por defecto (`NatNetwork`): `10.0.2.0/24`.
- La puerta de enlace siempre es la `.1` de la red (`10.0.2.1`) y el servidor DHCP la `.3` (`10.0.2.3`). Si creas la red `192.168.22.0/24`, la puerta de enlace será `192.168.22.1`.
- **No** se puede cambiar la IP de la puerta de enlace ni el rango que reparte el DHCP.

Ejemplo de una VM con Windows 7 en modo NAT Network:

![VM Windows 7 en modo NAT Network](https://www.nakivo.com/blog/wp-content/uploads/2019/07/A-Windows-7-VM-is-configured-to-work-in-the-NAT-Network-mode.webp)

**Comandos:**

```bash
# Crear una red NAT
VBoxManage natnetwork add --netname natnet1 --network "192.168.22.0/24" --enable

# Asignar el adaptador 1 de una VM a esa red
VBoxManage modifyvm "NombreVM" --nic1 natnetwork --nat-network1 natnet1
```

> ⚠️ Puede que tengas que **apagar la VM** antes de aplicar estos cambios.

**Diferencia clave con NAT:** las reglas de reenvío de puertos de NAT son **por VM** (`VM > Settings > Network`), mientras que las de NAT Network son **comunes a todas las VM** de esa red (`File > Preferences > Network`).

### 3.4 Bridged Adapter (adaptador puente)

La VM se conecta **directamente a tu red física**, como si fuera un equipo más enchufado al mismo switch. Los paquetes entran y salen de la tarjeta virtual sin pasar por ningún NAT.

- ✅ La VM ve al anfitrión, a los demás equipos de la LAN e Internet.
- ✅ El anfitrión y los demás equipos de la LAN **también ven a la VM**.
- ✅ Puede recibir IP del servidor DHCP de tu red física.

**Ideal para:** montar servidores en VM a los que deba acceder toda la red local.

**Ejemplo de direccionamiento:**

| Equipo | IP | Máscara | Puerta de enlace |
|---|---|---|---|
| Router / DHCP | `10.10.10.1` | | |
| Anfitrión | `10.10.10.72` | `255.255.255.0` | `10.10.10.1` |
| VM invitada | `10.10.10.91` | `255.255.255.0` | `10.10.10.1` |

Si el anfitrión tiene varias tarjetas (por ejemplo Ethernet y Wi-Fi), tienes que **elegir cuál usar** en el desplegable *Name*:

![Elegir el adaptador físico en modo Bridged](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-selecting-an-adapter-for-the-Bridged-network-mode.webp)

![Red en modo Bridged](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-bridged-networking.webp)

> ⚠️ **Puente sobre Wi-Fi:** la VM no podrá usar las funciones de bajo nivel de la tarjeta (elegir redes Wi-Fi, modo monitor, etc.); tendrá que usar la conexión Wi-Fi del anfitrión. Si necesitas esas funciones (por ejemplo en Kali Linux), usa un **adaptador Wi-Fi USB** con *USB pass-through*.

**Comando:**

```bash
VBoxManage modifyvm "NombreVM" --nic1 bridged --bridgeadapter1 "eth0"
```

#### Modo promiscuo (Promiscuous Mode)

Normalmente una tarjeta de red solo acepta las tramas dirigidas a su propia MAC (o de difusión). En **modo promiscuo** acepta **todo** el tráfico que le llegue. Es útil para análisis de red, auditorías de seguridad y para usar *sniffers* (como Wireshark) desde una VM.

Tiene tres opciones:

| Opción | Qué hace |
|---|---|
| **Deny** (por defecto) | La VM solo ve el tráfico dirigido a ella. |
| **Allow VMs** | La VM ve además el tráfico entre otras VM. |
| **Allow All** | Sin restricciones: la VM ve todo el tráfico. |

Se puede usar en los modos **Bridged, NAT Network, Internal Network y Host-only**. Ten en cuenta que la mayoría de tarjetas Wi-Fi **no soportan** modo promiscuo real.

### 3.5 Internal Network (red interna)

Las VM conectadas a la misma red interna forman una **red virtual totalmente aislada**: se ven entre ellas, pero **no** ven al anfitrión, ni a la LAN, ni a Internet. Tampoco se puede acceder a ellas desde fuera.

**Ideal para:** simular redes reales, montar laboratorios (routers, firewalls, servidores) sin riesgo de afectar a tu red física.

**Ejemplo de laboratorio con 3 VM:**

- **VM1** actúa como **router**: tiene dos tarjetas, una en red interna y otra en NAT.
- **VM2** y **VM3** solo tienen una tarjeta, en la red interna, y usan la VM1 como puerta de enlace para salir al exterior.

| VM | Red interna | NAT | Puerta de enlace |
|---|---|---|---|
| VM1 | `192.168.23.1` | `10.0.2.15` | `10.0.2.2` |
| VM2 | `192.168.23.2` | | `192.168.23.1` |
| VM3 | `192.168.23.3` | | `192.168.23.1` |

Subred interna: `192.168.23.0/24` (la defines tú manualmente, no hay DHCP por defecto).

![Red interna combinada con NAT](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-using-the-Internal-network-mode-in-a-combination-with-the-NAT-mode.webp)

> 💡 Para hacer de router en la VM1, lo habitual es instalar Linux y configurar `iptables`. Este mismo montaje sirve para **probar reglas de cortafuegos** antes de llevarlas a producción. Para conectar la VM1 con el exterior es preferible usar **Bridged** en lugar de NAT en su segunda tarjeta.

**Comando:**

```bash
VBoxManage modifyvm "NombreVM" --nic1 intnet --intnet1 "mi-red-interna"
```

Todas las VM que usen el mismo nombre (`mi-red-interna`) quedan en la misma red.

### 3.6 Host-only Adapter (solo anfitrión)

Crea una red privada **entre el anfitrión y las VM**. Las VM se ven entre sí y con el anfitrión, y el anfitrión ve a todas las VM. Pero **no hay salida a Internet ni a la LAN**.

![VM usando red Host-only](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-VMs-use-the-host-only-network.webp)

**Ideal para:** acceder desde tu equipo a servicios de la VM (SSH, web, bases de datos) sin exponerlos a la red. Se combina muy bien con una segunda tarjeta en NAT para que la VM también tenga Internet.

Se gestiona en **`File > Host Network Manager`**:

![Configurar la red Host-only](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-configuring-the-Host-Only-network.webp)

- Red por defecto: `192.168.56.0/24`.
- IP del anfitrión en esa red: `192.168.56.1`.
- Pestaña **Adapter**: cambias la IP y la máscara del adaptador virtual del anfitrión.
- Pestaña **DHCP Server**: activas o desactivas el DHCP y defines su IP, máscara y rango.

![Configurar el servidor DHCP de la red Host-only](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-configuring-a-DHCP-server-for-a-Host-Only-network.webp)

Detalles importantes:

- Las VM **no tienen puerta de enlace**, porque no hay nada fuera de esa red.
- Puedes crear **varias redes host-only** con el botón **Create**, y borrar las que no uses con **Remove**.

**Comando:**

```bash
VBoxManage modifyvm "NombreVM" --nic1 hostonly --hostonlyadapter1 "vboxnet0"
```

### 3.7 Generic Driver (controlador genérico)

Modo avanzado que permite compartir una interfaz de red usando controladores incluidos en VirtualBox o en su paquete de extensiones. Tiene dos submodos:

- **UDP Tunnel**: VM en **distintos anfitriones** se comunican de forma transparente usando la red existente.
- **VDE (Virtual Distributed Ethernet)**: conecta las VM a un switch virtual distribuido en anfitriones Linux o FreeBSD. Requiere **compilar VirtualBox desde el código fuente**, porque los paquetes estándar no lo incluyen.

Es un modo poco habitual; para la mayoría de laboratorios bastan los anteriores.

---

## 4. Tabla comparativa

Resumen de qué puede hacer cada modo:

| Modo | VM → anfitrión | VM ↔ VM | VM → Internet / LAN | Anfitrión → VM | LAN → VM |
|---|:---:|:---:|:---:|:---:|:---:|
| **Not attached** | ❌ | ❌ | ❌ | ❌ | ❌ |
| **NAT** | ✅ | ❌ | ✅ | Solo con port forwarding | Solo con port forwarding |
| **NAT Network** | ✅ | ✅ | ✅ | Solo con port forwarding | Solo con port forwarding |
| **Bridged** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Internal Network** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Host-only** | ✅ | ✅ | ❌ | ✅ | ❌ |

Versión visual del artículo original:

![Comparación de los modos de red de VirtualBox](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-Comparison-of-VirtualBox-Network-Modes.webp)

### ¿Cuál elijo?

| Necesito... | Modo recomendado |
|---|---|
| Que la VM solo tenga Internet | **NAT** |
| Varias VM que se vean entre sí y tengan Internet | **NAT Network** |
| Que la VM sea un equipo más de mi LAN (servidor accesible por todos) | **Bridged** |
| Un laboratorio totalmente aislado (routers, firewalls, pruebas) | **Internal Network** |
| Acceder yo a la VM desde mi equipo sin exponerla a la red | **Host-only** (+ NAT en un segundo adaptador si quiero Internet) |
| Simular un cable desconectado | **Not attached** o desmarcar *Cable Connected* |

---

## 5. Reenvío de puertos (Port Forwarding)

Por defecto, con **NAT** y **NAT Network** no se puede entrar a la VM desde fuera. El **reenvío de puertos** soluciona esto: VirtualBox escucha en un puerto del anfitrión y redirige el tráfico a un puerto de la VM.

**Cómo funciona por dentro:** el router NAT intercepta el paquete, lee la IP y el puerto de destino de las cabeceras y, si coinciden con una regla, reescribe esos datos y lo envía a la VM.

**Dónde se configura:**

| Modo | Ruta en la interfaz |
|---|---|
| NAT | `VM > Settings > Network > Advanced > Port Forwarding` (reglas **por VM**) |
| NAT Network | `File > Preferences > Network > editar la red > Port Forwarding` (reglas **compartidas**) |

Cada regla tiene estos campos: **Nombre**, **Protocolo** (TCP/UDP), **IP del host**, **Puerto del host**, **IP del invitado** y **Puerto del invitado**.

### Ejemplo 1: acceso SSH a una VM Ubuntu

**Datos del ejemplo:**

- IP del anfitrión: `10.10.10.72`
- IP de la VM (modo NAT): `10.0.2.15`
- Usuario: `user1`

**Paso 1: preparar el servidor SSH en la VM**

```bash
# Instalar el servidor SSH
sudo apt-get install openssh-server

# Editar la configuración
sudo vim /etc/ssh/sshd_config
```

Asegúrate de que esta línea está descomentada:

```
PasswordAuthentication yes
```

Reinicia el servicio y prueba en local:

```bash
sudo systemctl restart ssh
ssh user1@127.0.0.1
```

**Paso 2: crear la regla en VirtualBox**

Abre `Settings > Network`, elige el adaptador en NAT, despliega **Advanced**, pulsa **Port Forwarding** y añade una regla con el icono `+`.

![Configurar reenvío de puertos en modo NAT](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-configuring-port-forwarding-for-the-NAT-mode.webp)

SSH escucha por defecto en el puerto TCP 22. Creamos una regla para que el **puerto 8022 del anfitrión** apunte al **puerto 22 de la VM**.

**Opción A: solo accesible desde el propio anfitrión**

| Nombre | Protocolo | IP del host | Puerto del host | IP del invitado | Puerto del invitado |
|---|---|---|---|---|---|
| Ubuntu-SSH | TCP | 127.0.0.1 | 8022 | 10.0.2.15 | 22 |

![Regla SSH creada](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-the-SSH-port-forwarding-rule-is-created.webp)

Conéctate desde el anfitrión:

```bash
ssh -p 8022 user1@127.0.0.1
```

**Opción B: accesible desde otros equipos de la LAN**

Cambia la IP del host por la IP real de la tarjeta física del anfitrión:

| Nombre | Protocolo | IP del host | Puerto del host | IP del invitado | Puerto del invitado |
|---|---|---|---|---|---|
| Ubuntu-SSH | TCP | 10.10.10.72 | 8022 | 10.0.2.15 | 22 |

Conéctate desde cualquier equipo de la red:

```bash
ssh -p 8022 user1@10.10.10.72
```

> ⚠️ **Seguridad:** con la opción B expones un servicio de la VM a toda tu red. Usa contraseñas robustas o, mejor, autenticación por clave SSH.

### Ejemplo 2: acceso HTTP a un servidor web

**Paso 1: instalar Apache en la VM**

```bash
sudo apt-get install apache2
```

Si tienes un cortafuegos activo en la VM, permite el puerto TCP 80. Comprueba en el navegador de la VM que funciona abriendo `http://127.0.0.1` (debería verse la página por defecto de Apache).

**Paso 2: crear la regla**

En `VM settings > Network > [tu adaptador] > Port Forwarding` añade:

| Nombre | Protocolo | IP del host | Puerto del host | IP del invitado | Puerto del invitado |
|---|---|---|---|---|---|
| Ubuntu-HTTP80 | TCP | 10.10.10.72 | 8080 | 10.0.2.15 | 80 |

**Paso 3: probar**

Desde el anfitrión o cualquier equipo de la LAN, abre:

```
http://10.10.10.72:8080
```

![Regla HTTP creada correctamente](https://www.nakivo.com/blog/wp-content/uploads/2019/07/VirtualBox-network-settings-–-the-HTTP-port-forwarding-rule-has-been-created-successfully.webp)

Puedes crear reglas parecidas para RDP (3389), FTP (21) y cualquier otro protocolo.

### Reenvío de puertos por línea de comandos

```bash
# Añadir la regla SSH a una VM en modo NAT
VBoxManage modifyvm "NombreVM" --natpf1 "Ubuntu-SSH,tcp,127.0.0.1,8022,10.0.2.15,22"

# Borrar la regla
VBoxManage modifyvm "NombreVM" --natpf1 delete "Ubuntu-SSH"
```

---

## 6. Chuleta de comandos VBoxManage

```bash
# Ver las redes host-only, NAT y bridged disponibles
VBoxManage list hostonlyifs
VBoxManage list natnets
VBoxManage list bridgedifs

# Cambiar el modo de un adaptador
VBoxManage modifyvm "VM" --nic1 nat
VBoxManage modifyvm "VM" --nic1 natnetwork --nat-network1 natnet1
VBoxManage modifyvm "VM" --nic1 bridged --bridgeadapter1 "eth0"
VBoxManage modifyvm "VM" --nic1 intnet --intnet1 "mi-red-interna"
VBoxManage modifyvm "VM" --nic1 hostonly --hostonlyadapter1 "vboxnet0"
VBoxManage modifyvm "VM" --nic1 none

# Cambiar el modelo de tarjeta (ejemplo: virtio)
VBoxManage modifyvm "VM" --nictype1 virtio

# Modo promiscuo (deny | allow-vms | allow-all)
VBoxManage modifyvm "VM" --nicpromisc1 allow-all

# Conectar / desconectar el "cable"
VBoxManage modifyvm "VM" --cableconnected1 off
```

> Los cambios con `modifyvm` requieren que la VM esté **apagada**. El nombre exacto de las interfaces (`eth0`, `vboxnet0`...) cambia según tu sistema, así que compruébalo con los comandos `list`.

---

## 7. Conclusión

VirtualBox ofrece una configuración de red muy flexible:

- Hasta **8 adaptadores** por VM, cada uno con su **propio modo**.
- **6 modelos de tarjeta** emulados (AMD, Intel y virtio), con MAC configurable y opción de simular el cable conectado o desconectado.
- **Modos de red** para cada necesidad: NAT (Internet), NAT Network (VM conectadas entre sí), Bridged (integrada en tu LAN), Internal (laboratorio aislado), Host-only (acceso solo desde tu equipo) y Generic (casos avanzados).
- **Reenvío de puertos** para poder acceder a servicios de una VM detrás de NAT.

**Regla rápida:** empieza con **NAT** para tener Internet, añade **Host-only** cuando quieras entrar a la VM desde tu equipo, y usa **Internal Network** o **NAT Network** cuando montes laboratorios con varias máquinas.

---

*Fuente original: [NAKIVO – Configuración de redes de VirtualBox](https://www.nakivo.com/es/blog/virtualbox-network-setting-guide/).*
