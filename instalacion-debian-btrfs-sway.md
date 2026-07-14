# Instalación de Debian 13 (Trixie) con Btrfs + Sway

Guía práctica, paso a paso, para instalar y dejar funcionando un escritorio Debian 13 con Btrfs, Sway y audio vía PipeWire. Cada fase se prueba en hardware real antes de pasar a la siguiente.

**Estado de esta guía:** Fases 1-8 validadas en hardware real (incluyendo la corrección de PATH/variables y el terminal `foot` que faltaba en la Fase 7). Fases 9-11 (Bluetooth/monitores/snapshots, aplicaciones de escritorio, Wlogout) recién redactadas con paquetes confirmados contra los repositorios de Trixie — pendientes de validar en tu hardware. Fases 12 (personalización: menús, temas, atajos) y 13 (verificación final) siguen en discusión activa y no están escritas todavía.

**Nota de prioridad:** Bluetooth se dejó fuera de esta guía a propósito. Para el funcionamiento básico de la máquina, Internet importa más que Bluetooth, así que Bluetooth se atenderá junto con el resto de "integración de hardware" (Fase 9), *después* de tener ya red, escritorio y audio resueltos, no antes ni al mismo tiempo.

## Antes de empezar

- Todos los comandos de las Fases 1 a 5 se ejecutan como `root`, dentro o fuera del chroot según se indique.
- A partir de la Fase 6 (dentro del chroot todavía) y hasta la 8, sigues como `root` en el chroot. Ya arrancado el sistema instalado (fases futuras 9+), los comandos con `sudo` se ejecutan como usuario normal.
- El usuario a crear (`dilusaper`, "Diego Salazar" en este documento) se define como variable editable en la Fase 5, igual que `$DISK` en la Fase 1. No necesitas buscar y reemplazar nada a mano en el resto del documento.

### Si cierras la terminal a mitad de la instalación (antes de reiniciar)

Cada fase, desde la 3 en adelante, empieza restaurando `$PATH` y haciendo `source /root/instalacion.env`. Ese archivo se crea en la Fase 2 y guarda `$DISK`, `$EFI_PART`, `$SWAP_PART`, `$ROOT_PART` (y desde la Fase 5, también `$USER_NAME`/`$FULL_NAME`). Esto es intencional: si el live-USB se reinicia, la terminal se cierra, o simplemente vuelves otro día, **no necesitas repetir la Fase 1 completa ni recordar tus variables**. Basta con:

1. Volver a montar los subvolúmenes y hacer el bind-mount de `/dev`, `/proc`, `/sys`, `/run` (repite los comandos de montaje de las Fases 2 y 3, sin repetir `mkfs`/`btrfs subvolume create`/`debootstrap`).
2. Entrar de nuevo con `chroot /mnt /bin/bash`.
3. Ejecutar `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin && source /root/instalacion.env`.
4. Continuar desde la fase donde ibas.

Los pasos 1-3 son exactamente los del **Apéndice de rescate** al final de esta guía — es la misma operación, solo que la haces antes del primer reinicio en vez de después.

### Sobre el disco: HDD/USB (`sdX`) vs NVMe (`nvme0nX`)

Los discos NVMe (y eMMC) nombran sus particiones con una `p` antes del número (`nvme0n1p1`), mientras que los discos SATA/USB no la usan (`sda1`). Para no tener que editar cada comando de la guía según el disco, en la Fase 1 se define una detección automática y tres variables (`$EFI_PART`, `$SWAP_PART`, `$ROOT_PART`) que se usan durante el resto del documento en vez de `${DISK}1`, `${DISK}2`, `${DISK}3`. Si cambias de un HDD de pruebas a tu NVMe real, solo tienes que cambiar el valor de `$DISK`; el resto de la guía no necesita ningún ajuste.

------

## Fase 1: Preparación del disco

### Objetivo

Particionar el disco en EFI, swap y una partición Linux formateada en Btrfs.

### Qué logra esta fase

Deja el disco listo con tres particiones: EFI (arranque), swap (hibernación) y Btrfs (sistema), usando variables que funcionan igual en HDD/USB que en NVMe.

### Comandos a ejecutar

Primero identifica tu disco (no forma parte del bloque a pegar, solo es diagnóstico):

```bash
lsblk -p
sudo su
```

Ajusta esta variable a tu disco real antes de continuar (edítala, no la pegues tal cual):

```bash
export DISK="/dev/sdX"
```

El resto de este bloque ya se puede copiar y pegar sin más ediciones:

```bash
# Detecta si el disco necesita el sufijo "p" antes del número de partición
# (NVMe/eMMC) o no (SATA/USB tipo sdX). A partir de aquí, toda la guía usa
# $EFI_PART, $SWAP_PART y $ROOT_PART en vez de ${DISK}1/2/3.
case "$DISK" in
  *nvme*|*mmcblk*) PARTSEP="p" ;;
  *)               PARTSEP=""  ;;
esac
export EFI_PART="${DISK}${PARTSEP}1"
export SWAP_PART="${DISK}${PARTSEP}2"
export ROOT_PART="${DISK}${PARTSEP}3"

# Particionado GPT
sgdisk -Z $DISK                                    # limpia tabla de particiones existente
sgdisk -og $DISK                                   # crea tabla GPT nueva
sgdisk -n 1::+1G   -t 1:ef00 -c 1:'EFI'   $DISK
sgdisk -n 2::+24G  -t 2:8200 -c 2:'SWAP'  $DISK     # ajusta el tamaño según tu RAM si usarás hibernación
sgdisk -n 3::      -t 3:8300 -c 3:'LINUX' $DISK

# Formateo
mkfs.fat -F32 -n EFI $EFI_PART
mkswap -L SWAP $SWAP_PART
mkfs.btrfs -f -L DEBIAN $ROOT_PART
```

### Archivos modificados

Tabla de particiones de `$DISK`.

### Cómo verificar

```bash
lsblk -f $DISK
```

Deberías ver las tres particiones con sus labels (`EFI`, `SWAP`, `DEBIAN`) y sistemas de archivos correctos.

### Problemas comunes

- Si `echo $EFI_PART` no coincide con lo que ves en `lsblk -p` (por ejemplo, un disco poco común que no sea ni `sdX` ni `nvme`/`mmcblk`), ajusta `PARTSEP` manualmente antes de continuar.

------

## Fase 2: Subvolúmenes Btrfs

### Objetivo

Crear el layout de subvolúmenes y montarlos en su ubicación final.

### Qué logra esta fase

Separa `/`, `/home`, `/var/log`, `/var/cache`, `/var/tmp`, `/opt`, `/var/spool` y `/var/lib/libvirt` en subvolúmenes independientes, lo que facilita snapshots selectivos más adelante (Snapper solo necesita snapshotear `@` y `@home`).

### Comandos a ejecutar

```bash
mount -v $ROOT_PART /mnt

# Subvolúmenes
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@opt
btrfs subvolume create /mnt/@spool
btrfs subvolume create /mnt/@libvirt

umount -vR /mnt

export BTRFS_OPTS="defaults,noatime,space_cache=v2,compress=zstd:1"
mount -vo $BTRFS_OPTS,subvol=@ $ROOT_PART /mnt
mkdir -vp /mnt/{home,opt,boot/efi,var/{cache,lib/libvirt,log,spool,tmp}}

# Persiste las variables de disco en el propio sistema de destino. Al estar
# dentro de /mnt, sobrevive a que cierres esta terminal, reinicies el live-USB,
# o entres al chroot en otro momento: dentro del chroot esto será /root/instalacion.env.
mkdir -vp /mnt/root
cat > /mnt/root/instalacion.env << EOF
export DISK="$DISK"
export PARTSEP="$PARTSEP"
export EFI_PART="$EFI_PART"
export SWAP_PART="$SWAP_PART"
export ROOT_PART="$ROOT_PART"
export BTRFS_OPTS="$BTRFS_OPTS"
EOF

mount -vo $BTRFS_OPTS,subvol=@home    $ROOT_PART /mnt/home
mount -vo $BTRFS_OPTS,subvol=@opt     $ROOT_PART /mnt/opt
mount -vo $BTRFS_OPTS,subvol=@cache   $ROOT_PART /mnt/var/cache
mount -vo $BTRFS_OPTS,subvol=@libvirt $ROOT_PART /mnt/var/lib/libvirt
mount -vo $BTRFS_OPTS,subvol=@log     $ROOT_PART /mnt/var/log
mount -vo $BTRFS_OPTS,subvol=@spool   $ROOT_PART /mnt/var/spool
mount -vo $BTRFS_OPTS,subvol=@tmp     $ROOT_PART /mnt/var/tmp
mount -v $EFI_PART /mnt/boot/efi
```

### Archivos modificados

Ninguno fuera de `/mnt` (aún no es el sistema final).

### Cómo verificar

```bash
mount | grep /mnt
btrfs subvolume list /mnt
cat /mnt/root/instalacion.env
```

Deben aparecer los 8 subvolúmenes, todos los puntos de montaje bajo `/mnt`, y las cuatro variables con valores no vacíos (si alguna sale vacía aquí, corrígela ahora — es mucho más fácil que descubrirlo en la Fase 4).

### Problemas comunes

- Si olvidas `umount -vR /mnt` antes de remontar con subvolúmenes, el segundo `mount` de `@` puede fallar o montar el top-level en vez del subvolumen.

------

## Fase 3: Bootstrap de Debian

### Objetivo

Instalar el sistema base de Debian con `debootstrap` y entrar al chroot.

### Qué logra esta fase

Deja un sistema Debian mínimo instalado en `/mnt`, listo para configurarse desde dentro (chroot).

### Comandos a ejecutar

```bash
debootstrap --arch=amd64 trixie /mnt http://deb.debian.org/debian

for dir in dev proc sys run; do
    mount -v --rbind "/${dir}" "/mnt/${dir}"
    mount -v --make-rslave "/mnt/${dir}"
done
mount -v -t efivarfs efivarfs /mnt/sys/firmware/efi/efivars

chroot /mnt /bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export LANG=en_US.UTF-8
source /root/instalacion.env
```

### Archivos modificados

Todo el árbol de `/mnt` (nuevo sistema Debian).

### Cómo verificar

```bash
whoami          # debe responder "root"
cat /etc/debian_version
echo "$ROOT_PART / $EFI_PART / $SWAP_PART"   # no debe salir vacío
```

### Problemas comunes

- Si `debootstrap` falla a mitad de camino por red, vuelve a ejecutarlo: es reanudable.
- Las tres líneas después de `chroot` (`export PATH`, `export LANG`, `source /root/instalacion.env`) son las que garantizan que las Fases 4 a 8 funcionen aunque hayas cerrado la terminal antes de llegar aquí. **No te las saltes**, incluso si crees que las variables ya estaban puestas: no cuesta nada repetirlas y es la causa más común de que `blkid`, `swapon`, `useradd` o `grub-install` fallen más adelante con la partición vacía o con "command not found".

------

## Fase 4: Sistema base

### Objetivo

Configurar identidad del sistema (hostname, fstab, timezone, locale, teclado) e instalar el kernel y las herramientas base.

### Qué logra esta fase

Deja un sistema arrancable con el idioma y teclado correctos, fstab consistente con los subvolúmenes de la Fase 2, y el kernel + Btrfs + GRUB instalados. La red se configura en la Fase 6 (justo después de tener usuario y bootloader), no aquí, para mantener esta fase enfocada en lo estrictamente base.

### Comandos a ejecutar

```bash
# Por si volviste a entrar al chroot en una terminal nueva: restaura PATH y variables
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env

BTRFS_UUID=$(blkid -s UUID -o value $ROOT_PART)
EFI_UUID=$(blkid -s UUID -o value $EFI_PART)
SWAP_UUID=$(blkid -s UUID -o value $SWAP_PART)

# Corta aquí en vez de escribir un fstab con UUID= vacío si algo salió mal arriba
for v in BTRFS_UUID EFI_UUID SWAP_UUID; do
    [ -z "${!v}" ] && { echo "ERROR: $v está vacío. Revisa \$ROOT_PART/\$EFI_PART/\$SWAP_PART antes de seguir."; return 1 2>/dev/null || exit 1; }
done

cat > /etc/fstab << EOF
UUID=$BTRFS_UUID /                btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@ 0 0
UUID=$BTRFS_UUID /home            btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@home 0 0
UUID=$BTRFS_UUID /opt             btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@opt 0 0
UUID=$BTRFS_UUID /var/cache       btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@cache 0 0
UUID=$BTRFS_UUID /var/lib/libvirt btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@libvirt 0 0
UUID=$BTRFS_UUID /var/log         btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@log 0 0
UUID=$BTRFS_UUID /var/spool       btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@spool 0 0
UUID=$BTRFS_UUID /var/tmp         btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@tmp 0 0
UUID=$EFI_UUID   /boot/efi        vfat  defaults,noatime 0 2
UUID=$SWAP_UUID  none             swap  defaults 0 0
EOF

echo "debian" > /etc/hostname

cat > /etc/hosts << EOF
127.0.0.1       localhost
127.0.1.1       debian

::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
EOF

ln -sf /usr/share/zoneinfo/America/Bogota /etc/localtime

apt update
apt install -y locales console-setup

sed -i 's/^# *\(en_US.UTF-8\)/\1/' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/default/locale

cat > /etc/default/keyboard << EOF
XKBMODEL="pc105"
XKBLAYOUT="us"
XKBVARIANT="intl"
XKBOPTIONS=""
BACKSPACE="guess"
EOF
setupcon

cat > /etc/apt/sources.list << EOF
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian trixie main contrib non-free non-free-firmware

deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
deb-src http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
EOF

apt update

# Kernel, firmware genérico y herramientas base (la red se instala en la Fase 6)
apt install -y linux-image-amd64 linux-headers-amd64 \
    firmware-linux firmware-linux-nonfree \
    grub-efi-amd64 efibootmgr \
    btrfs-progs sudo vim bash-completion

swapon -a
```

### Archivos modificados

`/etc/fstab`, `/etc/hostname`, `/etc/hosts`, `/etc/localtime`, `/etc/locale.gen`, `/etc/default/locale`, `/etc/default/keyboard`, `/etc/apt/sources.list`.

### Cómo verificar

```bash
locale
cat /etc/fstab
dpkg -l | grep linux-image
```

### Problemas comunes

- Si `locale-gen` no genera `en_US.UTF-8`, revisa que la línea en `/etc/locale.gen` haya quedado descomentada.

------

## Fase 5: Usuario, hibernación y GRUB

### Objetivo

Configurar hibernación, crear el usuario y dejar GRUB instalado y arrancable.

### Qué logra esta fase

Al reiniciar, el sistema arranca directo a Debian con GRUB, con un usuario funcional con permisos de `sudo`, y con la hibernación apuntando correctamente a la partición swap.

### Comandos a ejecutar

Primero, **edita únicamente estas dos líneas** con tu nombre y usuario reales (igual que `$DISK` en la Fase 1 — este bloque no lleva nada más para que no tengas que revisar de más al editarlo):

```bash
export FULL_NAME="Diego Salazar"
export USER_NAME="dilusaper"
```

El resto se copia y pega tal cual, sin editar nada:

```bash
# Por si volviste a entrar al chroot en una terminal nueva: restaura PATH y variables
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env

echo "GRUB_CMDLINE_LINUX_DEFAULT=\"quiet resume=UUID=$SWAP_UUID\"" >> /etc/default/grub

cat > /etc/initramfs-tools/conf.d/resume << EOF
RESUME=UUID=$SWAP_UUID
EOF

update-initramfs -u -k all

# Se guardan junto al resto de variables para que las Fases 7 y 8 (y cualquier
# reingreso al chroot) los recuerden sin que tengas que volver a escribirlos
cat >> /root/instalacion.env << EOF
export FULL_NAME="$FULL_NAME"
export USER_NAME="$USER_NAME"
EOF

groupadd libvirt
useradd -m -G sudo,adm,libvirt -s /bin/bash -c "$FULL_NAME" $USER_NAME
passwd $USER_NAME

grub-install \
  --target=x86_64-efi \
  --efi-directory=/boot/efi \
  --bootloader-id=debian \
  --removable \
  --recheck

update-grub
```

### Archivos modificados

`/etc/default/grub`, `/etc/initramfs-tools/conf.d/resume`, `/boot/grub/grub.cfg`, `/etc/passwd` (nuevo usuario).

### Cómo verificar

```bash
grep resume /etc/default/grub
id $USER_NAME
ls /boot/efi/EFI
```

### Problemas comunes

- Si `grub-install` se queja de no encontrar `/boot/efi`, confirma que se montó en la Fase 2.
- Si `useradd`/`groupadd`/`grub-install`/`update-grub`/`update-initramfs` dan "command not found" aunque los paquetes estén instalados, tu `$PATH` no tiene `/sbin`/`/usr/sbin` en esta sesión: repite el `export PATH=...` del inicio de este bloque (revisa también que no hayas saltado el `source /root/instalacion.env` de la Fase 3).

------

## Fase 6: Redes (Internet)

### Objetivo

Dejar la conectividad de red (cableada y Wi-Fi) funcionando mediante NetworkManager, antes de instalar nada gráfico.

### Qué logra esta fase

Garantiza que la máquina instalada tenga Internet apenas arranque, incluso si por algún motivo las fases siguientes (escritorio, audio, etc.) quedaran a medias. Usa el firmware `iwlwifi` para tarjetas Intel.

### Comandos a ejecutar

```bash
# Por si volviste a entrar al chroot en una terminal nueva: restaura PATH y variables
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env

apt install -y network-manager wpasupplicant wireless-tools rfkill \
               firmware-iwlwifi

systemctl enable NetworkManager
```

### Archivos modificados

Ninguno relevante; NetworkManager gestiona su propia configuración en `/etc/NetworkManager/`.

### Cómo verificar

Tras el primer arranque (ver el corte "hasta aquí desde el live" más adelante):

```bash
nmcli device status
nmcli device wifi list
sudo nmtui   # conectar a una red desde una interfaz de texto simple
ping -c3 deb.debian.org
```

### Problemas comunes

- Si tu tarjeta Wi-Fi no aparece en `nmcli device status`, tu chipset puede necesitar un paquete de firmware distinto a `firmware-iwlwifi` (por ejemplo `firmware-realtek` o `firmware-atheros`); revisa el modelo con `lspci -k | grep -A3 -i network`.
- Si `rfkill list` muestra el Wi-Fi bloqueado por software, usa `rfkill unblock wifi`.

------

## Fase 7: Primer escritorio gráfico y navegador

### Objetivo

Instalar Sway, LightDM y Firefox ESR: lo mínimo necesario para tener un escritorio gráfico funcional, con acceso a Internet vía navegador, desde el cual continuar el resto de la instalación cómodamente.

### Qué logra esta fase

No es el escritorio final: es un punto de apoyo. Con esto ya puedes iniciar sesión gráficamente, navegar a GitHub para consultar el resto de tus notas/repositorios, y seguir instalando desde ahí en vez de depender de la consola o del chroot. También se establece la estructura modular de configuración (`config.d/`) que usarán las fases siguientes.

### Comandos a ejecutar

```bash
# Por si volviste a entrar al chroot en una terminal nueva: restaura PATH y variables
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env

apt install -y sway xwayland swaybg swaylock swayidle \
               foot \
               brightnessctl upower acpi \
               libinput-tools pciutils usbutils \
               lightdm lightdm-gtk-greeter \
               firefox-esr

systemctl enable lightdm.service

# Estructura modular de configuración de Sway
mkdir -p /home/$USER_NAME/.config/sway/config.d

cat > /home/$USER_NAME/.config/sway/config << 'EOF'
# Config principal: conserva los atajos por defecto de Sway/Debian
# y añade las personalizaciones propias en config.d/
include /etc/sway/config

include config.d/*.conf
EOF

cat > /home/$USER_NAME/.config/sway/config.d/variables.conf << 'EOF'
# Variables reutilizables por el resto de los archivos de config.d/
set $mod Mod4
# Terminal por defecto para $mod+Return. Se actualiza a kitty en la Fase 10;
# foot se mantiene instalado como respaldo aunque deje de ser el default.
set $term foot
EOF

cat > /home/$USER_NAME/.config/sway/config.d/inputs.conf << 'EOF'
# Teclado en inglés con AltGr para acentos (intl)
input * {
    xkb_layout "us"
    xkb_variant "intl"
}
EOF

cat > /home/$USER_NAME/.config/sway/config.d/outputs.conf << 'EOF'
# Configuración específica de monitores (resolución, escala, posición).
# Lista tus salidas con "wlr-randr" (Fase 8) y ajusta aquí según tu hardware.
EOF

cat > /home/$USER_NAME/.config/sway/config.d/bindings.conf << 'EOF'
# Atajos de teclado propios. Los de multimedia se añaden en la Fase 8.
EOF

cat > /home/$USER_NAME/.config/sway/config.d/appearance.conf << 'EOF'
# Tema Nord y apariencia general. Se completará en la fase de personalización.
EOF

cat > /home/$USER_NAME/.config/sway/config.d/autostart.conf << 'EOF'
# Aplicaciones que deben iniciar junto con la sesión (Waybar, etc.)
# Se completará en la fase de aplicaciones de escritorio.
EOF

cat > /home/$USER_NAME/.config/sway/config.d/idle.conf << 'EOF'
# Prueba rápida de gestión de energía (ajustar tiempos reales más adelante):
# Suspender a los 60s, hibernar a los 120s
exec swayidle -w \
    timeout 60  'systemctl suspend' \
    timeout 120 'systemctl hibernate'
EOF

chown -R $USER_NAME:$USER_NAME /home/$USER_NAME/.config
```

### Archivos modificados

`~/.config/sway/config` y `~/.config/sway/config.d/*.conf` (nuevos).

### Cómo verificar

Tras reiniciar, inicia sesión en LightDM eligiendo la sesión Sway. Deberías obtener un escritorio Sway vacío pero funcional, con el teclado en layout `us intl`. Presiona **`$mod+Return`** (`Super+Enter`) y debe abrirse una terminal `foot`; desde ahí, `firefox-esr &` debe abrir el navegador (usa la red configurada en la Fase 6).

### Problemas comunes

- Si LightDM no lista Sway como sesión, confirma que exista `/usr/share/wayland-sessions/sway.desktop` (lo instala el paquete `sway`).
- Si `include /etc/sway/config` da error porque el archivo fue modificado por otro paquete, revisa la ruta con `cat /etc/sway/config`.

------

## Fase 8: Audio, pantallas y teclado retroiluminado

### Objetivo

Dejar el audio completamente funcional a nivel de usuario, con pavucontrol como control gráfico, manejo básico de pantallas, y atajos de teclado seguros para volumen, mute de micrófono, brillo y retroiluminación del teclado.

### Qué logra esta fase

Instala PipeWire/WirePlumber junto con los componentes que, en pruebas sobre hardware real, resultaron necesarios para que el audio efectivamente se conecte a la sesión del usuario (bus de sesión, permisos de tiempo real y firmware/config de ALSA). Todo esto se hace todavía dentro del chroot: en vez de reiniciar servicios de usuario (lo cual no funciona sin una sesión real activa), se usa `systemctl --global enable`, que sí funciona sin sesión y deja los servicios listos para arrancar solos en el primer login real. También añade atajos multimedia que usan la lógica nativa de PipeWire (`wpctl`, no `pactl`), con un límite de volumen del 175% (en vez de 100%, para altavoces flojos) y un mínimo de brillo del 1% (para que la pantalla nunca se apague del todo).

### Comandos a ejecutar

```bash
# Por si volviste a entrar al chroot en una terminal nueva: restaura PATH y variables
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env

apt install -y pipewire wireplumber pipewire-pulse \
               dbus-user-session rtkit libpam-systemd \
               alsa-utils alsa-ucm-conf alsa-topology-conf firmware-sof-signed \
               pavucontrol \
               wlr-randr
# El usuario debe poder leer/escribir brillo sin sudo
usermod -aG video $USER_NAME
# Habilita los servicios de usuario de PipeWire para TODOS los usuarios,
# sin necesitar una sesión activa (no funciona "--user restart" en chroot).
# Al hacer login por primera vez tras reiniciar, ya arrancarán solos.
systemctl --global enable pipewire wireplumber pipewire-pulse
# Atajos multimedia, con límites para evitar los problemas conocidos:
# - wpctl con "-l 1.75" permite subir hasta 175% (ajusta a 2.0 = 200% si aún te
#   quedas corto de volumen; recuerda bajarlo tú mismo en espacios silenciosos).
# - brightnessctl con "--min-value=1" evita apagar la pantalla al bajar el brillo.
# - El nombre de dispositivo "tpacpi::kbd_backlight" es el habitual en ThinkPad
#   (driver thinkpad_acpi). Confirma con "brightnessctl -l" (Fase 8, verificación);
#   si tu equipo muestra otro nombre bajo la categoría "leds", ajústalo aquí.
#   Nota: en muchos ThinkPad, Fn+Espacio cicla el brillo del teclado directo por
#   firmware/EC, sin pasar por Sway, así que debería seguir funcionando igual
#   independientemente de este atajo (que es solo una vía alterna).
cat >> /home/$USER_NAME/.config/sway/config.d/bindings.conf << 'EOF'

# --- Multimedia: audio (PipeWire vía wpctl) ---
bindsym XF86AudioRaiseVolume exec wpctl set-volume -l 1.75 @DEFAULT_AUDIO_SINK@ 5%+
bindsym XF86AudioLowerVolume exec wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
bindsym XF86AudioMute        exec wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
bindsym XF86AudioMicMute     exec wpctl set-mute @DEFAULT_AUDIO_SOURCE@ toggle

# --- Multimedia: brillo de pantalla ---
bindsym XF86MonBrightnessUp   exec brightnessctl set 5%+
bindsym XF86MonBrightnessDown exec brightnessctl set 5%- --min-value=1

# --- Multimedia: retroiluminación del teclado (ThinkPad: tpacpi::kbd_backlight) ---
bindsym XF86KbdBrightnessUp   exec brightnessctl -d tpacpi::kbd_backlight set 10%+
bindsym XF86KbdBrightnessDown exec brightnessctl -d tpacpi::kbd_backlight set 10%-
EOF

chown $USER_NAME:$USER_NAME /home/$USER_NAME/.config/sway/config.d/bindings.conf
```

### Archivos modificados

`~/.config/sway/config.d/bindings.conf` (atajos multimedia añadidos).

### Cómo verificar

Después de reiniciar y hacer login gráfico:

```bash
# Servicios de usuario activos (deberían arrancar solos por el --global enable)
systemctl --user status pipewire wireplumber pipewire-pulse

# El audio ve un sink y una fuente por defecto
wpctl status

# Prueba de sonido
speaker-test -c2 -twav -l1

# Control gráfico de volumen
pavucontrol

# Salidas de pantalla detectadas
wlr-randr
```

Prueba las teclas de volumen/brillo/retroiluminación:

- El volumen no debe superar el 175% aunque mantengas presionada la tecla de subir.
- El brillo no debe llegar a 0% aunque mantengas presionada la tecla de bajar.

### Problemas comunes

- Si tras el primer login `systemctl --user status pipewire` no aparece activo, ejecútalo manualmente una vez: `systemctl --user restart pipewire wireplumber pipewire-pulse` (esto solo funciona con sesión real, por eso no se hizo en el chroot).
- Si `wpctl status` no muestra ningún sink, revisa que no quede `pulseaudio` instalado en paralelo (compite por el mismo socket que `pipewire-pulse`).
- Si `brightnessctl` pide permisos, confirma que el usuario pertenece al grupo `video` (`groups $USER_NAME`) y reinicia sesión para que el grupo tome efecto.
- Si `brightnessctl -l` muestra el LED del teclado con un nombre distinto a `tpacpi::kbd_backlight`, corrige las dos líneas de `bindings.conf` con el nombre real antes de reiniciar.
- En portátiles con audio Intel (HDA/SoundWire) que no reproducen nada, `firmware-sof-signed` suele ser la pieza faltante; revisa `dmesg | grep -i sof` tras instalarlo.
- Si `brightnessctl -l` no muestra ningún dispositivo de tipo "leds", tu equipo probablemente no tiene teclado retroiluminado controlable por software; puedes borrar esas dos líneas de `bindings.conf`.

------

## ✅ HASTA AQUÍ DESDE EL LIVE — sistema mínimo funcional

Con las Fases 1 a 8 completas (todavía dentro del chroot desde el entorno live), ya tienes: disco particionado y montado, sistema base, usuario y GRUB, red, escritorio gráfico con Firefox ESR, y audio/pantallas/teclado retroiluminado configurados. A partir de aquí **ya no necesitas seguir en el chroot**: reinicias, entras a tu escritorio real, y usas el propio sistema instalado (con su navegador y su red ya funcionando) para continuar con las fases futuras (Bluetooth, Snapper/Btrfs Assistant, Waybar, tema Nord, aplicaciones adicionales, verificación final).

```bash
exit          # sale del chroot
umount -vR /mnt
reboot
```

------

## Fase 9: Bluetooth, monitores y snapshots (integración de hardware)

*Ya en el sistema real, arrancado desde el disco — no es chroot.*

### Objetivo

Dejar Bluetooth funcional con gestor gráfico, herramientas de gestión de monitores/color, y las herramientas de snapshots instaladas (sin política automática todavía).

### Qué logra esta fase

Bluetooth emparejable desde una interfaz gráfica, monitores gestionables gráficamente y con perfiles guardables, un perfil de color por defecto como red de seguridad, y Snapper/Btrfs Assistant listos para que definas tu propia política de snapshots cuando quieras (no antes).

### Comandos a ejecutar

```bash
sudo apt update
sudo apt install -y bluez bluez-tools blueman
sudo systemctl enable --now bluetooth.service
```

```bash
# kanshi: guarda/aplica perfiles de monitores automáticamente (útil si conectas
# un externo). nwg-displays: GUI para acomodar monitores sin editar texto.
# colord + gnome-color-manager: gestión de color con un perfil sRGB por defecto
# como red de seguridad si una calibración manual futura sale mal.
sudo apt install -y kanshi nwg-displays colord gnome-color-manager
```

```bash
# Solo se instalan las herramientas. La política de snapshots (qué, cuándo,
# cuántas conservar) se configura manualmente más adelante, a propósito.
sudo apt install -y snapper btrfs-assistant
```

### Archivos modificados

Ninguno de configuración todavía; solo paquetes instalados y el servicio `bluetooth` habilitado.

### Cómo verificar

```bash
bluetoothctl show                 # el adaptador debe aparecer "Powered: yes"
blueman-manager                   # abre la GUI; deberías poder buscar dispositivos
nwg-displays                      # abre la GUI de monitores
dpkg -L gnome-color-manager | grep bin/   # confirma el binario real antes de lanzarlo
btrfs-assistant                   # abre y debe listar tus subvolúmenes @, @home, etc.
```

### Problemas comunes

- Si `blueman-manager` no encuentra dispositivos, confirma `rfkill list` (el Bluetooth no debe aparecer "Soft blocked").
- Si `btrfs-assistant` pide permisos, ejecútalo con `sudo` o autentica vía el diálogo de Polkit que debería aparecer.

------

## Fase 10: Aplicaciones de escritorio

*Sistema real.*

### Objetivo

Kitty como terminal principal, Nemo como explorador de archivos, Flatpak con los portales necesarios y Bitwarden, soporte completo de AppImage, y Synaptic como gestor de paquetes gráfico.

### Qué logra esta fase

Un escritorio con las aplicaciones diarias resueltas con software maduro, sin scripts propios de por medio.

### Comandos a ejecutar

```bash
sudo apt install -y kitty nemo
```

Actualiza la terminal por defecto de `$mod+Return` (definida en la Fase 7) para que ahora abra Kitty; `foot` se queda instalado como respaldo:

```bash
sed -i 's/^set \$term foot$/set $term kitty/' ~/.config/sway/config.d/variables.conf
```

```bash
# Flatpak + portales necesarios para que apps en sandbox (Bitwarden incluido)
# abran diálogos nativos y funcionen bien en Wayland/Sway sin escritorio completo.
sudo apt install -y flatpak xdg-desktop-portal xdg-desktop-portal-wlr xdg-desktop-portal-gtk

flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install -y flathub com.bitwarden.desktop
```

```bash
# Soporte base para ejecutar AppImages (la mayoría siguen enlazados a FUSE2)
sudo apt install -y libfuse2 desktop-file-utils shared-mime-info

# AppImageLauncher no está en los repos de Debian: se baja el .deb más reciente
# publicado en GitHub. Verifica en https://github.com/TheAssassin/AppImageLauncher/releases
# que el patrón de nombre de archivo siga siendo el mismo antes de confiar en el grep.
AIL_URL=$(curl -s https://api.github.com/repos/TheAssassin/AppImageLauncher/releases/latest \
  | grep "browser_download_url.*bullseye_amd64.deb" | cut -d '"' -f4)
wget -O /tmp/appimagelauncher.deb "$AIL_URL"
sudo apt install -y /tmp/appimagelauncher.deb
```

```bash
sudo apt install -y synaptic
```

### Archivos modificados

`~/.config/sway/config.d/variables.conf` (terminal por defecto).

### Cómo verificar

```bash
kitty --version
nemo                                    # abre el explorador de archivos
flatpak run com.bitwarden.desktop &     # Bitwarden debe abrir con diálogos nativos
synaptic                                # gestor gráfico de paquetes
```

Descarga cualquier `.AppImage` de prueba, dale doble clic desde Nemo: AppImageLauncher debe ofrecer integrarlo (crear su entrada en el menú) automáticamente.

### Problemas comunes

- Si el `grep` del `.deb` de AppImageLauncher no encuentra nada, revisa manualmente la página de releases: puede que hayan cambiado el nombre del asset (por ejemplo, a otra versión base de Ubuntu/Debian).
- Si Bitwarden (Flatpak) no abre el selector de archivos con el estilo esperado, confirma que `xdg-desktop-portal-gtk` quedó instalado y reinicia sesión.

------

## Fase 11: Gestión de sesión (Wlogout)

*Sistema real.*

### Objetivo

Un único punto gráfico para bloquear, suspender, cerrar sesión, reiniciar o apagar — sin duplicar esa lógica en ningún otro menú.

### Comandos a ejecutar

```bash
sudo apt install -y wlogout
```

```bash
# Reemplaza el atajo de salida de Sway por defecto (que solo confirma "salir de
# Sway") para que en su lugar abra Wlogout con todas las opciones.
cat >> ~/.config/sway/config.d/bindings.conf << 'EOF'

# --- Gestión de sesión ---
bindsym $mod+Shift+e exec wlogout
EOF
```

### Cómo verificar

```bash
wlogout   # deben aparecer los íconos de bloquear/suspender/cerrar sesión/reiniciar/apagar
```

### Problemas comunes

- Si los íconos de Wlogout salen sin estilo (cuadros de texto en vez de íconos), es normal en su tema por defecto hasta que lo personalicemos en la fase de estética.

------

Si necesitas arrancar desde un live USB para reparar el sistema instalado (por ejemplo, GRUB roto o un initramfs mal generado), estos son los pasos para volver a entrar por chroot.

Primero identifica tu disco (diagnóstico, no forma parte del bloque a pegar):

```bash
lsblk -p
```

Ajusta esta variable a tu disco real (edítala, no la pegues tal cual):

```bash
export DISK="/dev/sdX"
```

El resto se puede copiar y pegar sin más ediciones:

```bash
case "$DISK" in
  *nvme*|*mmcblk*) PARTSEP="p" ;;
  *)               PARTSEP=""  ;;
esac
export EFI_PART="${DISK}${PARTSEP}1"
export SWAP_PART="${DISK}${PARTSEP}2"
export ROOT_PART="${DISK}${PARTSEP}3"
export BTRFS_OPTS="defaults,noatime,space_cache=v2,compress=zstd:1"

mount -vo $BTRFS_OPTS,subvol=@ $ROOT_PART /mnt

mount -vo $BTRFS_OPTS,subvol=@home    $ROOT_PART /mnt/home
mount -vo $BTRFS_OPTS,subvol=@opt     $ROOT_PART /mnt/opt
mount -vo $BTRFS_OPTS,subvol=@cache   $ROOT_PART /mnt/var/cache
mount -vo $BTRFS_OPTS,subvol=@libvirt $ROOT_PART /mnt/var/lib/libvirt
mount -vo $BTRFS_OPTS,subvol=@log     $ROOT_PART /mnt/var/log
mount -vo $BTRFS_OPTS,subvol=@spool   $ROOT_PART /mnt/var/spool
mount -vo $BTRFS_OPTS,subvol=@tmp     $ROOT_PART /mnt/var/tmp
mount -v $EFI_PART /mnt/boot/efi

for dir in dev proc sys run; do
    mount -v --rbind "/${dir}" "/mnt/${dir}"
    mount -v --make-rslave "/mnt/${dir}"
done
mount -v -t efivarfs efivarfs /mnt/sys/firmware/efi/efivars

chroot /mnt /bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env   # recupera también $USER_NAME/$FULL_NAME si ya los definiste
```

**Sobre qué hacer una vez dentro:** esto solo te deja *entrar* al sistema instalado; no implica que siempre debas regenerar GRUB o el initramfs. Son ejemplos de las tareas típicas que resuelves desde aquí, condicionadas al problema que estés diagnosticando:

- `update-grub` — solo si GRUB no arranca o no detecta el kernel (por ejemplo, tras un cambio de disco/partición o una actualización de kernel hecha sin regenerar el menú).
- `update-initramfs -u -k all` — solo si cambiaste algo que afecta el arranque temprano (módulos, configuración de resume/hibernación, disco).

Si solo entraste a revisar un archivo de configuración o los logs, no necesitas ejecutar ninguno de los dos.

```bash
update-grub
update-initramfs -u -k all
```

Para salir limpiamente:

```bash
exit
umount -vR /mnt
reboot
```
