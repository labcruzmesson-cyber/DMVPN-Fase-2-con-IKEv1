# VPN Hub and Spoke Punto a Multipunto DMVPN Fase 2 con IKEv1 y Enrutamiento Dinámico

**Asignatura:** Seguridad de Redes
**Estudiante:** Manuel Alejandro Cruz Messon
**Matrícula:** 2025-0689
**Docente:** Jonathan Rondón
**Fecha:** 2 de julio de 2026

> **Nota:** el direccionamiento no se realizó en base a la matrícula del estudiante; este requerimiento se recordó demasiado tarde y no fue posible rehacer las topologías y videos.

---

## Tabla de Contenidos

- [1. Resumen y Objetivos](#1-resumen-y-objetivos)
- [2. Topología de Red y Direccionamiento](#2-topología-de-red-y-direccionamiento)
- [3. Especificación de Políticas y Parámetros de Seguridad (ISAKMP / IKEv1)](#3-especificación-de-políticas-y-parámetros-de-seguridad-isakmp--ikev1)
- [4. Configuración de la Nube DMVPN (mGRE + NHRP)](#4-configuración-de-la-nube-dmvpn-mgre--nhrp)
- [5. Puntos Críticos Analizados](#5-puntos-críticos-analizados)
- [6. Protocolo de Verificación y Diagnóstico Técnico](#6-protocolo-de-verificación-y-diagnóstico-técnico)

---

## 1. Resumen y Objetivos

Este documento describe la evolución del laboratorio hacia una arquitectura **DMVPN (Dynamic Multipoint VPN) Fase 2**, en topología Hub-and-Spoke con tres sedes: PEER A actuando como **Hub (NHS - Next Hop Server)** y PEER B / PEER C actuando como **Spokes**. Se introduce el uso de interfaces **GRE Multipunto (mGRE)** sobre el protocolo **NHRP (Next Hop Resolution Protocol)** para el registro dinámico de los spokes, protegidas mediante IPsec en modo transporte con Fase 1 basada en **ISAKMP (IKEv1)**, y se habilita el protocolo de enrutamiento dinámico **EIGRP** para la propagación automática de las rutas LAN entre sedes. Adicionalmente, se configura **NAT dinámico con sobrecarga (PAT)** en cada peer para permitir la salida a Internet del tráfico no destinado al túnel.

### Objetivos del Proyecto

- **Arquitectura Hub-and-Spoke Escalable (DMVPN Fase 2):** configurar PEER A como Next Hop Server (NHS) mediante una interfaz `Tunnel0` en modo `gre multipoint`, permitiendo que múltiples spokes se registren dinámicamente sin necesidad de túneles punto a punto individuales.
- **Enrutamiento Dinámico entre Sedes:** habilitar EIGRP sobre la nube DMVPN (red `192.168.100.0/24`) para la propagación automática de las rutas `172.16.1.0/24`, `172.16.2.0/24` y `172.16.3.0/24` entre los tres peers, evitando el enrutamiento estático manual.
- **Confidencialidad e Integridad del Túnel:** garantizar la protección del tráfico inter-sede mediante ISAKMP (IKEv1) con clave precompartida comodín (wildcard, `address 0.0.0.0`) y un transform-set IPsec en modo transporte, indispensable para permitir múltiples spokes bajo la misma política criptográfica.
- **Salida a Internet Independiente (NAT Overload):** permitir que el tráfico de cada LAN local salga hacia Internet mediante NAT dinámico con sobrecarga (PAT) por interfaz física, de forma independiente al tráfico cifrado que circula por la nube DMVPN.

---

## 2. Topología de Red y Direccionamiento

La topología física crece de dos a tres sedes interconectadas mediante un gateway WAN que emula un proveedor de servicios de Internet (R-ISP). PEER A funge como Hub (NHS) de la nube DMVPN, mientras que PEER B y PEER C se registran como Spokes apuntando hacia la dirección de túnel de PEER A (`192.168.100.1`).

| Dispositivo / Sede | Interfaz | Dirección IP | Máscara de Subred | Propósito / Rol |
|---|---|---|---|---|
| PEER A (HUB) | Gi0/0 | 10.0.0.60 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER A (HUB) | Gi0/1 | 172.16.1.1 | 255.255.255.0 | Gateway LAN A / NAT Inside |
| PEER A (HUB) | Tunnel0 | 192.168.100.1 | 255.255.255.0 | NHS DMVPN / Hub mGRE |
| PEER B (SPOKE) | Gi0/0 | 10.0.0.70 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER B (SPOKE) | Gi0/1 | 172.16.2.1 | 255.255.255.0 | Gateway LAN B / NAT Inside |
| PEER B (SPOKE) | Tunnel0 | 192.168.100.2 | 255.255.255.0 | Spoke DMVPN registrado al Hub |
| PEER C (SPOKE) | Gi0/0 | 10.0.0.80 | 255.255.255.0 | Enlace WAN (IP Pública) / NAT Outside |
| PEER C (SPOKE) | Gi0/1 | 172.16.3.1 | 255.255.255.0 | Gateway LAN C / NAT Inside |
| PEER C (SPOKE) | Tunnel0 | 192.168.100.3 | 255.255.255.0 | Spoke DMVPN registrado al Hub |
| R-ISP (Gateway) | N/A | 10.0.0.1 | 255.255.255.0 | Puerta de enlace predeterminada WAN |

---

## 3. Especificación de Políticas y Parámetros de Seguridad (ISAKMP / IKEv1)

Al tratarse de una arquitectura multipunto con un número variable de spokes, la Fase 1 se configura mediante una clave precompartida comodín (wildcard key), asociada a la dirección `0.0.0.0`, permitiendo que cualquier peer remoto autentique bajo la misma clave sin necesidad de declarar cada IP de forma individual.

### Configuración NAT y ACL de Salida a Internet — PEER A

El NAT dinámico con sobrecarga (overload/PAT) se aplica sobre toda la LAN local mediante una ACL permisiva, permitiendo la salida a Internet. Al no existir aquí una exclusión explícita del tráfico inter-LAN (NAT bypass), es la propia lógica de enrutamiento hacia `Tunnel0` la que evita que el tráfico destinado a las otras sedes sea traducido, dado que dicho tráfico será enrutado por EIGRP hacia la interfaz de túnel en lugar de hacia Gi0/0.

### Fase 1: ISAKMP Policy y Wildcard Pre-Shared Key

| Parámetro | Valor |
|---|---|
| Algoritmo de Cifrado | AES (128 bits por defecto) |
| Integridad y Hash | SHA-1 (hash sha), para los procesos de autenticación e intercambio seguro |
| Grupo Diffie-Hellman | Grupo 2 (Exponenciación modular de 1024 bits) para la generación de claves efímeras |
| Autenticación | Llave precompartida comodín (wildcard), asociada a la dirección `0.0.0.0`, lo cual permite que cualquier peer (Hub o Spoke) autentique bajo la misma clave `DMVPN_KEY` sin declaraciones individuales por IP |
| Pre-Shared Key | `DMVPN_KEY` |

### Fase 2: IPsec Transform Set en Modo Transporte

| Parámetro | Valor |
|---|---|
| Nombre del Set | `TS_DMVPN` |
| Encapsulación Criptográfica | ESP con AES (128 bits por defecto, esp-aes) |
| Autenticación/Hashing de Datos | ESP bajo código de autenticación de mensajes con SHA-1 (esp-sha-hmac) |
| Modo de Operación | Modo Transporte (mode transport), requerido en DMVPN dado que la cabecera de enrutamiento externa ya es provista por el encapsulamiento mGRE |
| Perfil IPsec | `PROF_DMVPN`, aplicado sobre la interfaz `Tunnel0` de cada peer mediante `tunnel protection`, sin asociarse a un perfil IKEv2 (se utiliza el modelo ISAKMP global) |

---

## 4. Configuración de la Nube DMVPN (mGRE + NHRP)

El elemento central de esta arquitectura es la interfaz `Tunnel0` configurada en modo `gre multipoint`, la cual permite el establecimiento dinámico de túneles GRE hacia múltiples peers sin necesidad de una interfaz dedicada por cada spoke. El protocolo **NHRP (Next Hop Resolution Protocol)** es responsable de la resolución dinámica entre la dirección de túnel (NBMA lógica) y la dirección IP física real de cada peer.

- **PEER A — Hub (NHS)** — Interfaz Tunnel0 y EIGRP

  `no ip split-horizon eigrp 1` y `no ip next-hop-self eigrp 1` son obligatorios en el Hub para permitir que las rutas aprendidas de un spoke sean reenviadas hacia los demás spokes conservando el next-hop original, habilitando así la comunicación spoke-to-spoke sobre EIGRP.

- **PEER B — Spoke** — Interfaz Tunnel0 y EIGRP

- **PEER C — Spoke** — Interfaz Tunnel0 y EIGRP

---

## 5. Puntos Críticos Analizados

### A. DMVPN Fase 2: Túneles Dinámicos Spoke-to-Hub

A diferencia de las variantes GRE punto a punto anteriores (donde cada `Tunnel0` declaraba un único `tunnel destination` fijo), en DMVPN Fase 2 la interfaz opera en modo `gre multipoint`, sin destino fijo. Cada spoke se registra dinámicamente ante el Hub mediante NHRP, declarando la dirección del NHS (`ip nhrp nhs`) y su correspondencia NBMA (`ip nhrp map`). En Fase 2, la comunicación entre spokes debe pasar obligatoriamente por el Hub (spoke-hub-spoke), ya que los spokes no resuelven túneles directos entre sí.

### B. Rol Crítico del Hub en EIGRP (Split-Horizon y Next-Hop-Self)

El Hub debe deshabilitar `split-horizon` y `next-hop-self` de EIGRP sobre la interfaz `Tunnel0`; de lo contrario, las rutas anunciadas por un spoke no serían reenviadas hacia los demás spokes, o se reenviarían con un next-hop inalcanzable (la propia IP de túnel del Hub), rompiendo la conectividad LAN-a-LAN entre PEER B y PEER C.

### C. Convivencia de NAT Overload con el Tráfico DMVPN

El NAT dinámico con sobrecarga se configura sobre toda la LAN local sin una exclusión explícita del tráfico inter-sede. Esto es consistente en DMVPN, ya que el tráfico hacia las redes remotas (`172.16.2.0/24`, `172.16.3.0/24`, etc.) es dirigido automáticamente hacia `Tunnel0` por la tabla de enrutamiento generada por EIGRP, antes de evaluarse contra la política de NAT asociada a la interfaz física Gi0/0 — a diferencia del tráfico hacia Internet, que sí sale traducido por dicha interfaz.

---

## 6. Protocolo de Verificación y Diagnóstico Técnico

### Paso 1: Verificación del Registro NHRP de los Spokes en el Hub

Antes de generar tráfico de datos, es indispensable confirmar que los spokes se hayan registrado exitosamente ante el NHS. Este comando se ejecuta en PEER A (Hub):

```
PEER_A# show ip nhrp
```

**Criterio de Aceptación:** deben listarse las entradas dinámicas correspondientes a `192.168.100.2` (PEER B) y `192.168.100.3` (PEER C), cada una asociada a su dirección NBMA real (`10.0.0.70` y `10.0.0.80` respectivamente).

### Paso 2: Generación de Tráfico Interesante para Levantamiento de Túnel

Se emite una solicitud de eco ICMP desde un host terminal interno de cada LAN spoke con destino a la LAN de otra sede, forzando el levantamiento de las asociaciones de seguridad hacia el Hub.

### Paso 3: Validación del Canal de Control en Fase 1 (ISAKMP SA)

Para examinar el estado del intercambio de llaves entre el Hub y cada spoke, se emplea el comando de diagnóstico:

```
Router# show crypto isakmp sa
```

**Criterio de Aceptación:** el resultado en consola debe declarar de forma explícita el estado **QM_IDLE** por cada peer activo. En el Hub (PEER A) deben observarse tantas asociaciones de seguridad como spokes registrados y comunicándose (hasta dos, una por cada spoke activo).
