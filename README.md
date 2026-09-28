# Conectar Wi-Fi en Arch Linux desde la terminal (nmcli)

_Guardado: 2026-09-28 · Probado en BlackArch (host `blackarch`), tarjeta `wlp9s0`, NetworkManager_

En este equipo la red la maneja **NetworkManager**, así que la herramienta de terminal es **`nmcli`**.
(`iwctl` no está instalado: es de `iwd`, otra alternativa que aquí no se usa. No mezclar ambos.)

---

## Paso 0 — Confirmar que NetworkManager está activo

```bash
systemctl status NetworkManager
```
- Debe decir `active (running)`.
- Si no: `sudo systemctl enable --now NetworkManager`
  (`enable` = arranca solo al encender; `--now` = arráncalo ya).

## Paso 1 — Ver las interfaces de red

```bash
nmcli device status
```
Ejemplo en este equipo:
```
DEVICE   TYPE      STATE          CONNECTION
enp8s0   ethernet  connected      Wired connection 1
wlp9s0   wifi      disconnected   --
lo       loopback  connected (externally)  lo
```
- `wlp9s0` es la tarjeta Wi-Fi (`w` = wireless, `p9s0` = posición en el bus PCI).
- `enp8s0` es la tarjeta cableada (Ethernet).

## Paso 2 — Comprobar que el Wi-Fi no esté apagado/bloqueado

```bash
nmcli radio wifi     # debe decir: enabled
rfkill list          # "Soft blocked: no" y "Hard blocked: no"
```
- Si `nmcli radio wifi` dice `disabled` → `nmcli radio wifi on`
- **Soft blocked: yes** (bloqueo por software) → `rfkill unblock wifi`
- **Hard blocked: yes** (bloqueo físico) → tecla Fn / interruptor del portátil o BIOS.

## Paso 3 — Buscar redes disponibles

```bash
nmcli device wifi rescan     # fuerza un escaneo nuevo
nmcli device wifi list
```
Columnas importantes:

| Columna  | Qué indica |
|----------|------------|
| `IN-USE` | `*` = la red a la que estás conectado |
| `SSID`   | Nombre de la red |
| `BSSID`  | MAC del punto de acceso (un mismo SSID puede tener varios, ej. router 2.4 GHz y 5 GHz) |
| `CHAN`   | Canal (1–13 = 2.4 GHz, 36+ = 5 GHz) |
| `SIGNAL` | Intensidad de 0 a 100 |
| `SECURITY` | `WPA2`, `WPA3`, `--` = abierta (sin contraseña, evitar) |

## Paso 4 — Conectarse a la red

**Forma recomendada** (la contraseña NO queda en el historial de bash):
```bash
nmcli --ask device wifi connect "NOMBRE_DE_LA_RED"
```
Te pedirá la contraseña de forma oculta.

Forma rápida (⚠ la contraseña queda guardada en `~/.bash_history`):
```bash
nmcli device wifi connect "NOMBRE_DE_LA_RED" password "CONTRASEÑA"
```

- Las comillas son necesarias si el nombre tiene espacios (ej. `"Mi Red 5G"`).
- Si ves `Device 'wlp9s0' successfully activated` → conectado.
- Para elegir una interfaz concreta: añadir `ifname wlp9s0` al final.

**Red oculta** (no aparece en la lista):
```bash
nmcli --ask device wifi connect "NOMBRE_OCULTO" hidden yes
```

## Paso 5 — Verificar la conexión

```bash
nmcli device status          # wlp9s0 → connected
ip -br addr show wlp9s0      # IP que te dio el router por DHCP (ej. 192.168.1.x/24)
ip route                     # "default via 192.168.1.1 dev wlp9s0" = tu gateway
ping -c 3 192.168.1.1        # llegas al router (LAN)
ping -c 3 8.8.8.8            # llegas a internet (WAN)
ping -c 3 archlinux.org      # funciona el DNS (nombre → IP)
```
Si `8.8.8.8` responde pero `archlinux.org` no → problema de **DNS**, no de Wi-Fi.

> Si tienes cable **y** Wi-Fi conectados a la vez, `ip route` mostrará dos rutas `default`;
> gana la de menor `metric` (normalmente la cableada, 100, frente a Wi-Fi, 600).

## Paso 6 — Gestionar redes guardadas

NetworkManager guarda cada red como una "conexión" (perfil) y se reconecta sola.

```bash
nmcli connection show                          # todas las redes guardadas
nmcli connection show --active                 # solo las activas
nmcli connection up   "NOMBRE_DE_LA_RED"       # conectar a una red ya guardada
nmcli connection down "NOMBRE_DE_LA_RED"       # desconectar
nmcli device disconnect wlp9s0                 # desconectar la tarjeta Wi-Fi
nmcli connection delete "NOMBRE_DE_LA_RED"     # olvidar la red
nmcli connection modify "NOMBRE_DE_LA_RED" connection.autoconnect no   # que no se conecte sola
```
Los perfiles se guardan en `/etc/NetworkManager/system-connections/` (solo root puede leerlos
porque contienen la contraseña).

Ver la contraseña de una red guardada:
```bash
nmcli --show-secrets connection show "NOMBRE_DE_LA_RED" | grep psk
```

## Paso 7 — Alternativa con menú en la terminal

```bash
nmtui
```
Interfaz de texto con flechas y Enter: *Activate a connection* → elegir red → contraseña.

---

## Resumen rápido

```bash
nmcli radio wifi on
nmcli device wifi rescan
nmcli device wifi list
nmcli --ask device wifi connect "MiRed"
nmcli device status
ip -br addr show wlp9s0 && ping -c 3 8.8.8.8
```

## Problemas comunes

| Síntoma | Causa / solución |
|---------|------------------|
| `No network with SSID ... found` | Nombre mal escrito (distingue mayúsculas) → `nmcli device wifi rescan` y copiar el SSID exacto |
| `Secrets were required, but not provided` | Contraseña incorrecta → `nmcli connection delete "Red"` y volver a conectar |
| `wlp9s0` no aparece | Falta el driver/firmware → `lspci -k | grep -A3 -i network` y `sudo pacman -S linux-firmware` |
| `unavailable` en `nmcli device` | Wi-Fi bloqueado → Paso 2 (`rfkill unblock wifi`) |
| Conecta pero no hay internet | Revisar `ip route` (gateway) y DNS: `resolvectl status` o `cat /etc/resolv.conf` |

## Relación con el estudio de redes / Nmap

- Al conectar, el router te asigna IP, máscara, gateway y DNS por **DHCP** (tema 6 de la Tercera Clase de Nmap).
- La MAC de `wlp9s0` en este equipo (`ca:d1:8e:...`) tiene `a` como segundo dígito → es una **MAC aleatoria**
  (bit *locally administered*): NetworkManager la cambia por privacidad al escanear/conectar.
