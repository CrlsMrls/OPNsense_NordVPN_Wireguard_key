
# OPNsense + NordVPN (WireGuard)

Este documento describe cómo crear una interfaz WireGuard (NordLynx) en OPNsense usando una subscripción de NordVPN.

English translation available [here](README.md)

- [Guía de Configuración: OPNsense + NordVPN (WireGuard)](#guía-de-configuración-opnsense--nordvpn-wireguard)
  - [0. Introducción](#0-introducción)
  - [1. Requisitos Previos](#1-requisitos-previos)
    - [Paso 1.1: Obtener tu Clave Privada (Private Key)](#paso-11-obtener-tu-clave-privada-private-key)
    - [Paso 1.2: Obtener datos del Servidor](#paso-12-obtener-datos-del-servidor)
  - [2. Configuración de WireGuard en OPNsense](#2-configuración-de-wireguard-en-opnsense)
    - [Paso 2.1: Configurar tu Conexión (Instance)](#paso-21-configurar-tu-conexión-instance)
    - [Paso 2.2: Configurar el Servidor Remoto (Peer)](#paso-22-configurar-el-servidor-remoto-peer)
    - [Paso 2.3: Asociar Peer a Instance](#paso-23-asociar-peer-a-instance)
    - [Paso 2.4: Activar el Peer](#paso-24-activar-el-peer)
    - [Paso 2.5: Verificar Conexión](#paso-25-verificar-conexión)


Versiones:
*   OPNsense 25
*   NordVPN Subscripción Activa (2026)

## 0. Introducción

Ni NordVPN ni OPNsense ofrecen una guía oficial unificada para esta configuración.

- NordVPN ofrece WireGuard bajo el nombre de "NordLynx", su implementación personalizada para mayor velocidad y privacidad. Sin embargo, a diferencia de otros proveedores, no facilitan archivos de configuración .conf ni muestran la Clave Privada en el panel de usuario, lo que dificulta la configuración manual en routers de terceros.

- OPNsense tiene soporte nativo para WireGuard, pero carece de una guía específica para NordVPN. Además, la documentación oficial sobre WireGuard suele estar desactualizada respecto a los cambios recientes en la interfaz de usuario (como el cambio de terminología a "Instances" y "Peers" en la versión 25.7).


## 1. Requisitos Previos

Antes de configurar OPNsense, necesitas obtener tus credenciales y claves desde un terminal (macOS/Linux).

### Paso 1.1: Obtener tu Clave Privada (Private Key)
Por algún motivo NordVPN oculta esto, es posible que sólo quieran que usemos sus aplicaciones. 

Navega a "Access Tokens" y crea uno nuevo con permisos para "Obtener credenciales de servicio".
1.  [NordAccount](https://my.nordaccount.com/dashboard/) > 
2.  [NordVPN](https://my.nordaccount.com/dashboard/nordvpn/) >
3.  [Access token](https://my.nordaccount.com/dashboard/nordvpn/access-tokens/) >
4.  You will need to verify your email address.
5.  Crea un nuevo token con el permiso "Get service credentials". Asumamos que el token generado es `e9f2___X___66`.
6.  El siguiente paso es usar ese token para obtener tu Private Key de WireGuard. Ejecuta este comando en tu terminal (reemplaza `YOUR_TOKEN_HERE` con tu token real):

```bash
curl -s -u token:YOUR_TOKEN_HERE https://api.nordvpn.com/v1/users/services/credentials | jq -r .nordlynx_private_key
Vnn______________fMQ=
```

**Nota:** Tienes que tener instalado `jq` para procesar JSON. Si no lo tienes, instálalo con `brew install jq` (macOS) o `sudo apt install jq` (Linux).

El comando anterior retorna tu **Private Key** para WireGuard.

### Paso 1.2: Obtener datos del Servidor 
Ejecuta esto para obtener la IP y Clave Pública del mejor servidor, en mi caso en España (ID 69):

```bash
curl -s "https://api.nordvpn.com/v1/servers/recommendations?filters\[country_id\]=202&limit=1" | jq -r '.[0] | {hostname: .hostname, ip: .station, public_key: (.technologies[] | select(.identifier=="wireguard_udp") | .metadata[] | select(.name=="public_key") | .value)}'

{
  "hostname": "es225.nordvpn.com",
  "ip": "185.214.97.110",
  "public_key": "OaSGa___________AXY="
}
```


## 2. Configuración de WireGuard en OPNsense

Para crear una conexión a WireGuard es necesario crear un "Instance" (tu conexión) y un "Peer" (el servidor remoto).


### Paso 2.1: Configurar tu Conexión (Instance)
Ve a **VPN > WireGuard > Instances** y añade una nueva:

*   **Name:** `NordVPN_Local`
*   **Private Key:** La clave `Vnn______________fMQ=` que obtuviste en el paso 1.1.
*   **Public Key:** Déjalo vacío, se calcula sola.
*   **Listen Port:** `51820`
*   **DNS Server:** `103.86.96.100` DNS de NordVPN.
*   **Tunnel Address:** `10.5.0.2/32` IP interna que se usará como como Gateway NordLynx.
*   **Disable Routes:** **MARCADO ✅** En mi caso no quiero que todo el tráfico del router pase por la VPN, solo el de un VLAN específica. Si quieres que TODO el tráfico del router pase por la VPN, déjalo sin marcar.
*   **Save**
*   **Apply**


### Paso 2.2: Configurar el Servidor Remoto (Peer)
Ve a **VPN > WireGuard > Peers** y añade uno nuevo:

*   **Name:** `NordVPN_ES_225` en este caso quiero identificador del servidor de España 225.
*   **Public Key:** La `public_key` del servidor que obtuviste en el paso 1.2, e.g `OaSGa___________AXY=`).
*   **Endpoint Address:** La `ip` numérica del servidor que obtuviste en el paso 1.2, e.g. `185.214.9.110`).
*   **Endpoint Port:** `51820`
*   **Allowed IPs:** `0.0.0.0/0` Permitir todo el tráfico de internet.
*   **Keepalive Interval:** `25` Set persistent keepalive interval in seconds.
*   **Save**
*   **Apply**

### Paso 2.3: Asociar Peer a Instance
Ve a **VPN > WireGuard > Instances** y linka el servidor remoto a tu conexión:

*  Edita la instancia `NordVPN_Local`.
*  En la pestaña **Peers**, añade el peer `NordVPN_ES_225`.
*   **Save**
*   **Apply**

### Paso 2.4: Activar el Peer
Ve a **VPN > WireGuard > Peers**:

* Edita el peer `NordVPN_ES_225`, click in pencil icon.
* **Enable:** Check ✅ This will enable the peer.
*   **Save**
*   **Apply**

### Paso 2.5: Verificar Conexión
Ve a **VPN > WireGuard > Status**:

Debe mostrarse la interface y el peer activo. Ambos con ✅ check verde.

OPNsense debería haber creado la interfaz `wg0` automáticamente, la interface y el peer usan esta interfaz. Si todo ha ido bien, el peer muestra tráfico TX/RX y la duración desde el último "Handshake".

En caso de problemas, revisa los logs en **VPN > WireGuard > Log File**.

