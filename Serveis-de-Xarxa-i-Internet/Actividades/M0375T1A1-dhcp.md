# Trabajar con el DHCP — Kea DHCP

**Módulo:** M0375 / Bastionado
**Ciclo:** ASIX
**Objetivo:** Trabajar el DHCP, analizar su negociación con Wireshark y migrar de ISC-DHCP a Kea DHCP.

**Datos de la red de ejemplo usados en esta guía:**
- Master (DNS + DHCP): `172.16.0.2`
- Esclavo (DNS): `172.16.0.3`
- Zona / dominio: `asix.com`
- Subred DHCP: `192.168.6.0/24`
- Interfaz de Red Interna: `enp0s8`

---

## Parte práctica — Negociación DHCP con Wireshark

### 1. Captura en tu PC (host)

Abre Wireshark en tu equipo, selecciona la interfaz de red activa (WiFi o Ethernet) y aplica el filtro:

```
bootp
```

> El protocolo DHCP se identifica internamente en Wireshark como `bootp` por herencia histórica de BOOTP. También puedes probar el filtro `dhcp`.

Para forzar una negociación nueva:

**Windows:**
```
ipconfig /release
ipconfig /renew
```

**Linux:**
```bash
sudo dhclient -r
sudo dhclient
```

Deberías ver los 4 paquetes del proceso **DORA**: `Discover`, `Offer`, `Request`, `Ack`.

> 📸 Captura de pantalla: Wireshark mostrando los 4 paquetes DORA.

### 2. Captura en la VM servidor + cliente

Repite el mismo proceso, pero ahora desde el equipo cliente conectado a tu servidor Kea DHCP (una vez instalado más abajo) y con Wireshark capturando en la interfaz de Red Interna.

> 📸 Captura de pantalla: negociación DHCP completa entre cliente y servidor Kea.

### 3. Dos capturas mostrando tu servidor de DNS

Haz dos capturas de Wireshark donde se vea tráfico hacia tu servidor DNS (`172.16.0.2` o `172.16.0.3`), por ejemplo una consulta DNS normal y una transferencia de zona AXFR/IXFR entre master y esclavo.

> 📸 Captura 1: consulta DNS al servidor.
> 📸 Captura 2: tráfico de sincronización entre master y esclavo (filtro `dns` o `dns.flags.opcode == 5` para NOTIFY).

### 4. Puertos que usa el protocolo DHCP

```bash
cat /etc/services | grep "boot"
cat /etc/services | grep "dhcp"
```

Resultado esperado y qué información muestra:

| Servicio | Puerto | Información |
|---|---|---|
| `bootps` | 67/udp | Puerto del **servidor** DHCP (recibe Discover/Request) |
| `bootpc` | 68/udp | Puerto del **cliente** DHCP (recibe Offer/Ack) |

> 📸 Captura de pantalla con la salida de ambos comandos.

### 5. Equipos clientes conectados

```bash
dhcp-lease-list
```

> Esta herramienta está pensada para **ISC-DHCP**, ya que lee `/var/lib/dhcp/dhcpd.leases`. Con Kea el fichero de concesiones es distinto (`/var/lib/kea/kea-leases4.csv`), así que el comando equivalente en Kea es:

```bash
cat /var/lib/kea/kea-leases4.csv
```

Esto muestra: dirección IP asignada, dirección MAC del cliente, tiempo de inicio y expiración de la concesión (`lease`), y el estado (activa, expirada, liberada).

> 📸 Captura de pantalla con el contenido del fichero de leases.

---

## Transición de ISC-DHCP a Kea DHCP

El servidor DHCP tradicional de ISC (`dhcpd`) alcanzó su fin de ciclo de vida oficial (**EOL**) a finales de 2022. Su sucesor, **Kea DHCP**, fue diseñado desde cero para redes empresariales e infraestructuras de ISPs actuales.

---

## Parte 1 — Arquitectura y procesos

### ¿De qué manera la arquitectura modular de Kea supera el modelo monolítico de ISC-DHCPd?

ISC-DHCPd es un **proceso único (monolítico)**: un solo demonio (`dhcpd`) gestiona la configuración, el reparto de IPv4 e IPv6, el logging y la gestión del servicio. Si ese proceso falla o se bloquea, se pierde todo el servicio DHCP a la vez.

Kea divide las funciones en **demonios independientes**:

| Demonio | Función |
|---|---|
| `kea-dhcp4` | Gestiona exclusivamente el protocolo DHCPv4 |
| `kea-dhcp6` | Gestiona exclusivamente el protocolo DHCPv6 |
| `kea-ctrl-agent` | Expone la API REST de control |
| `kea-dhcp-ddns` | Gestiona las actualizaciones dinámicas de DNS (DDNS) |

**Aislamiento de fallos:** si `kea-dhcp-ddns` falla, `kea-dhcp4` sigue repartiendo IPs con normalidad, algo imposible en el modelo monolítico de ISC-DHCPd.

**Escalabilidad:** al ser procesos separados, cada demonio puede reiniciarse, actualizarse o incluso desplegarse en distinto hardware de forma independiente, y se pueden asignar recursos (CPU, hilos) de forma específica a cada uno según la carga.

### ¿Qué función cumple `kea-ctrl-agent`?

Es el demonio que expone una **API REST sobre HTTP/JSON**, actuando de intermediario entre el administrador (o una herramienta externa) y los demonios DHCP.

En ISC-DHCPd, cualquier cambio de configuración requiere editar `dhcpd.conf` y enviar una señal de proceso como `SIGHUP` (recarga) o `SIGTERM` (parada), lo cual es poco flexible y difícil de automatizar.

Con `kea-ctrl-agent`, el administrador (o un script, un panel web, una herramienta de automatización) envía comandos JSON por HTTP, por ejemplo:

```bash
curl -X POST http://172.16.0.2:8000/ \
  -H "Content-Type: application/json" \
  -d '{ "command": "config-reload", "service": ["dhcp4"] }'
```

Esto permite gestión remota, integración con otras herramientas y automatización real, sin necesidad de acceso por SSH al servidor.

### ¿Cómo implementa Kea el multithreading y el procesamiento asíncrono?

Kea utiliza un **pool de hilos** para procesar las peticiones DHCP entrantes en paralelo, en lugar de atenderlas una a una de forma secuencial como hace ISC-DHCPd (que es esencialmente un proceso de un solo hilo para el procesamiento de paquetes).

El procesamiento de I/O (recepción de paquetes de red, consultas a la base de datos de leases) se gestiona de forma **asíncrona**: mientras un hilo espera la respuesta de una base de datos, otros hilos siguen procesando nuevas peticiones DHCP, evitando que el servidor se bloquee esperando operaciones de entrada/salida. Esto reduce drásticamente el cuello de botella en escenarios de alta concurrencia, como una renovación masiva de leases al reiniciarse una red corporativa grande.

---

## Parte 2 — Configuración

### Ventajas del formato JSON frente a `dhcpd.conf`

| Aspecto | `dhcpd.conf` (ISC-DHCP) | Kea (JSON) |
|---|---|---|
| **Sintaxis** | Propietaria, basada en bloques `{}` sin un estándar formal | JSON estándar, ampliamente soportado |
| **Validación** | Errores solo se detectan al recargar el servicio | Se puede validar con cualquier parser JSON antes de aplicar, o con `kea-dhcp4 -t` |
| **Automatización (IaC)** | Requiere plantillas de texto frágiles (regex, Jinja2) | Ansible y Terraform tienen módulos nativos para generar y manipular JSON |
| **Parsing** | Requiere un parser específico del lenguaje de ISC | Cualquier lenguaje de programación tiene librerías JSON nativas |

El uso de JSON hace que la configuración de Kea sea tratable como **código estructurado**, lo que encaja directamente con pipelines de Infraestructura como Código (IaC), donde la configuración se versiona, se valida automáticamente en CI/CD y se despliega de forma reproducible.

### Actualización de subredes en caliente sin reiniciar el servicio

Kea permite modificar la configuración (subredes, rangos, reservas) **en tiempo de ejecución**, enviando la nueva configuración a través de la API REST de `kea-ctrl-agent`:

```bash
curl -X POST http://172.16.0.2:8000/ \
  -H "Content-Type: application/json" \
  -d '{ "command": "config-set", "service": ["dhcp4"], "arguments": { ... } }'
```

El demonio aplica los cambios sin interrumpir las concesiones activas ni cerrar el socket de escucha.

**Comparación con ISC-DHCPd:** en el modelo tradicional, cualquier cambio en `dhcpd.conf` exige un `service isc-dhcp-server restart`, lo que corta momentáneamente el servicio. En un entorno de producción con miles de clientes, ese reinicio provoca que, durante la ventana de caída, los equipos que estén renovando su lease en ese instante no reciban respuesta, generando un pico de peticiones reintentadas al volver el servicio. Con Kea, ese impacto operacional desaparece.

---

## Parte 3 — Persistencia, almacenamiento y alta disponibilidad (HA)

### Diferencias entre `dhcpd.leases` y los backends de Kea

| Aspecto | `dhcpd.leases` (texto plano) | Backends de Kea (SQL/NoSQL) |
|---|---|---|
| **Rendimiento** | Lectura/escritura secuencial en fichero; lento con miles de leases | Consultas indexadas, mucho más rápidas a gran escala |
| **Recuperación ante fallos** | El fichero puede corromperse si el proceso se cae a mitad de escritura | Las bases de datos (PostgreSQL, MySQL) ofrecen transacciones ACID |
| **Consistencia** | Un solo fichero, un solo servidor; difícil de compartir entre nodos | Una base de datos centralizada puede ser consultada por varios servidores Kea a la vez (HA real) |

Además, Kea soporta el backend en memoria **Memfile** (CSV, el usado por defecto y en esta guía) para entornos pequeños, y backends en bases de datos reales (PostgreSQL, MySQL) o NoSQL (Cassandra) para entornos de producción de gran escala.

### Host Reservations Database y la integración con IPAM

Guardar las reservas de host en una **base de datos centralizada** en lugar de en el fichero de configuración estático tiene dos ventajas clave:

- Permite que **herramientas externas de IPAM** (gestión de direccionamiento IP) lean y escriban reservas directamente en la base de datos, sin tocar ningún fichero de configuración ni reiniciar el servicio DHCP.
- Evita el problema de ISC-DHCPd, donde añadir una reserva nueva implica editar `dhcpd.conf` manualmente y reiniciar el servicio, lo cual no es viable en redes con miles de reservas gestionadas por varios equipos a la vez.

### Failover de ISC-DHCP vs HA Hook Extension de Kea

| Aspecto | Failover (ISC-DHCP) | HA Hook Extension (Kea) |
|---|---|---|
| **Protocolo** | Protocolo propietario de ISC específico para DHCP | API REST estándar sobre HTTP |
| **Modos soportados** | Solo Activo-Standby | **Activo-Pasivo** y **Activo-Activo** |
| **Sincronización** | Mensajes binarios propios del protocolo failover | Los servidores se sincronizan vía API REST y comparten estado de leases |
| **Modo Activo-Activo** | No soportado | Ambos servidores pueden repartir IPs simultáneamente, duplicando capacidad y repartiendo carga |

El modo **Activo-Activo** de Kea es la diferencia más relevante: en ISC-DHCP, el servidor secundario permanece inactivo esperando un fallo del primario (desperdiciando capacidad), mientras que en Kea ambos nodos trabajan a la vez, mejorando tanto la disponibilidad como el rendimiento.

---

## Parte 4 — Automatizaciones y monitorización

### El framework de Hooks en Kea

Los **Hooks** son un sistema de extensión que permite añadir funcionalidad al núcleo de Kea **sin modificar el código fuente**. Son bibliotecas compartidas (`.so`) que se cargan en tiempo de ejecución y se "enganchan" a puntos concretos del flujo de procesamiento del paquete DHCP (por ejemplo, justo antes de asignar una IP, o justo después de recibir un Discover).

Casos de uso típicos:

- **Autenticación externa / integración con RADIUS:** un hook puede consultar un servidor RADIUS antes de conceder una IP, validando si el cliente tiene permiso de red.
- **Filtrado personalizado de paquetes:** un hook puede inspeccionar campos específicos de la petición DHCP (como la opción 82, información del relay) y decidir si responde o descarta la petición.

Esto evita tener que parchear o recompilar el núcleo de Kea cada vez que se necesita una funcionalidad específica de una organización, algo que en ISC-DHCPd solo era posible modificando directamente el código fuente en C.

### Métricas expuestas vía API JSON y su integración con herramientas modernas

Kea expone estadísticas en tiempo real a través de la misma API REST de `kea-ctrl-agent`, por ejemplo:

```bash
curl -X POST http://172.16.0.2:8000/ \
  -H "Content-Type: application/json" \
  -d '{ "command": "statistic-get-all", "service": ["dhcp4"] }'
```

Esto devuelve métricas como paquetes recibidos, leases asignados, leases declinados, tiempos de respuesta, etc., **en formato JSON estructurado**.

**Comparación con syslog tradicional:**

| Aspecto | Logs syslog (ISC-DHCPd) | API JSON (Kea) |
|---|---|---|
| **Formato** | Texto plano no estructurado | JSON estructurado, parseable directamente |
| **Integración con Prometheus/Grafana** | Requiere parsers (regex) frágiles para extraer métricas | Se puede exportar directamente a un exporter de Prometheus |
| **Integración con ELK Stack** | Logstash necesita reglas grok complejas | Ingesta directa del JSON sin transformación compleja |
| **Tiempo real** | Hay que esperar a que se escriba la línea en el log | Consulta directa al estado actual vía API |

En resumen, Kea permite una observabilidad moderna (dashboards en tiempo real, alertas automáticas) sin depender del parsing de texto plano que exige ISC-DHCPd.

---

## Parte 5 — Práctica y estrategias de migración

### `kea-admin` y la migración de reglas complejas (opción 60/43)

`kea-admin` es la herramienta de línea de comandos que gestiona las bases de datos de Kea (inicializar el esquema, actualizarlo entre versiones, hacer backups del backend de leases).

Para la migración de reglas de filtrado complejas, como las **clases de clientes basadas en las opciones 60/43** (identificación de fabricante y suboption del vendor, usado por ejemplo para distinguir teléfonos VoIP o puntos de acceso WiFi), el proceso no es automático: no existe una conversión 1:1 automática de `dhcpd.conf` a JSON para esta lógica concreta.

El administrador tiene que:

1. Identificar las condiciones de clase (`class` en ISC-DHCP, basada en `option vendor-class-identifier`)
2. Recrearlas manualmente en Kea usando **Client Classification**, definiendo expresiones con el lenguaje de evaluación de Kea, por ejemplo:

```json
"client-classes": [
    {
        "name": "VOIP_PHONES",
        "test": "option[60].text == 'VoIPPhone'"
    }
]
```

`kea-admin` por sí solo no convierte esta lógica; su función se limita a la gestión de la base de datos, no a la conversión de reglas de configuración.

### Escenario de pruebas de carga con `perfdhcp`

Diseño de un escenario de prueba para medir el **TPS** (transacciones por segundo) y el tiempo de respuesta al otorgar **10.000 concesiones simultáneas**:

**Entorno:**
- 1 VM con Kea DHCP (servidor bajo prueba), en la red `192.168.6.0/24`
- 1 VM con `perfdhcp` (cliente generador de carga), ambas en red interna aislada para no afectar a tráfico real

**Comando de prueba:**

```bash
sudo perfdhcp -4 -r 500 -R 10000 -D 10000 192.168.6.2
```

- `-4` → modo DHCPv4
- `-r 500` → tasa de 500 peticiones por segundo
- `-R 10000` → simula 10.000 clientes distintos
- `-D 10000` → envía 10.000 Discover en total
- `192.168.6.2` → IP del servidor Kea a probar

**Métricas a recoger:**
- TPS real alcanzado (comparado con el objetivo de 500/s)
- Tiempo medio y máximo de respuesta por transacción completa (DORA)
- Porcentaje de peticiones sin respuesta (timeout) o leases rechazados

**Análisis esperado:** gracias al multithreading y procesamiento asíncrono descritos en la Parte 1, Kea debería mantener un TPS más estable y un tiempo de respuesta más bajo que ISC-DHCPd en el mismo escenario, especialmente a medida que aumenta la concurrencia.

---

## Parte 6 — Instalación de Kea DHCP

### Requisitos previos

- VM con Ubuntu Server (en este caso, el **master**, `172.16.0.2`, donde ya está instalado el DNS)
- Dos adaptadores de red: uno NAT/Puente (acceso a internet para instalar paquetes) y otro Red Interna (`enp0s8`, donde se repartirán las IPs)

### Paso 1 — Actualizar e instalar

```bash
sudo apt update
sudo apt install kea-dhcp4-server kea-ctrl-agent -y
```

A diferencia de `isc-dhcp-server`, Kea instala **paquetes separados** por demonio, reflejando la arquitectura modular explicada en la Parte 1.

### Paso 2 — Copia de seguridad antes de editar

```bash
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.BKP
```

### Paso 3 — Configurar `kea-dhcp4.conf`

```bash
sudo nano /etc/kea/kea-dhcp4.conf
```

```json
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": [ "enp0s8" ]
    },
    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/kea-leases4.csv"
    },
    "valid-lifetime": 3600,
    "subnet4": [
        {
            "subnet": "192.168.6.0/24",
            "pools": [ { "pool": "192.168.6.10 - 192.168.6.100" } ],
            "option-data": [
                { "name": "routers", "data": "192.168.6.1" },
                { "name": "domain-name-servers", "data": "172.16.0.2" },
                { "name": "domain-name", "data": "asix.com" }
            ],
            "reservations": [
                {
                    "hw-address": "00:0C:27:C9:XX:XX",
                    "ip-address": "192.168.6.155"
                }
            ]
        }
    ]
}
}
```

> `domain-name-servers` apunta a `172.16.0.2`, el propio master, que ya hace de servidor DNS, cerrando el ciclo DHCP → cliente obtiene DNS → cliente resuelve en `asix.com`.

### Paso 4 — Validar y arrancar el servicio

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

Si no da errores de sintaxis:

```bash
sudo systemctl enable kea-dhcp4-server
sudo systemctl start kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

### Paso 5 — Comprobar desde un cliente

```
ipconfig /release
ipconfig /renew
```

Verificar en Wireshark (filtro `bootp`) que se completa el proceso DORA, y comprobar la IP asignada con `ipconfig /all` (Windows) o `ip a` (Linux).

### Paso 6 — Consultar las concesiones activas

```bash
cat /var/lib/kea/kea-leases4.csv
```

### Incidencias técnicas habituales

- **APIPA (169.254.x.x):** el cliente no ha podido contactar con el servidor Kea. Comprobar que `kea-dhcp4-server` está activo (`systemctl status`) y que la interfaz configurada en `interfaces-config` es la correcta.
- **Error de sintaxis JSON:** Kea es muy estricto con el formato JSON (comas, llaves). Usar siempre `kea-dhcp4 -t` antes de reiniciar el servicio para validar la configuración.
- **Conflicto con otro servidor DHCP:** si el switch/router de la red interna tiene su propio DHCP activo, los clientes pueden recibir una IP de ese dispositivo en lugar de la del servidor Kea. Desactivar el DHCP del router en la red interna de laboratorio.
- **Puerto 67/68 bloqueado por firewall:** comprobar con `sudo ss -lun | grep 67` que Kea está escuchando correctamente en el puerto 67/UDP.

> 📸 Captura de pantalla: `systemctl status kea-dhcp4-server` en estado `active (running)`.
> 📸 Captura de pantalla: contenido de `kea-leases4.csv` con al menos una concesión activa.
