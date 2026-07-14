# Comandos — Instalación Debian 13 (Btrfs + Sway)

Chuleta de solo comandos, derivada de `instalacion-debian-btrfs-sway.md` (esa es la referencia con explicaciones; si algo cambia, se edita ahí y se regenera este archivo).

Recordatorios rápidos:
- Fases 1-2: fuera del chroot, como root en el entorno live.
- Fase 3: entra al chroot. Fases 4-8: dentro del chroot.
- Fases 9 en adelante: ya en el sistema real, arrancado desde el disco (no chroot).
- Cada fase desde la 3 (mientras se está en el chroot) empieza restaurando `$PATH` y con `source /root/instalacion.env`.
- Ajusta `$DISK` (Fase 1), `$HOSTNAME` (Fase 4) y `$FULL_NAME`/`$USER_NAME` (Fase 5) a tu caso antes de pegar esos bloques — cada uno vive en su propio contenedor aislado, sin nada más alrededor.

------

## Fase 1: Preparación del disco

```bash
lsblk -p
sudo su
```

```bash
export DISK="/dev/sdX"
```

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

```bash
lsblk -f $DISK
```

------

## Fase 2: Subvolúmenes Btrfs

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
# Aparte de @, para que un rollback de la raíz nunca se lleve tus imágenes/
# contenedores por delante (Docker aún no se instala aquí, solo se reserva el subvolumen)
btrfs subvolume create /mnt/@docker

umount -vR /mnt

export BTRFS_OPTS="defaults,noatime,space_cache=v2,compress=zstd:1"
mount -vo $BTRFS_OPTS,subvol=@ $ROOT_PART /mnt
mkdir -vp /mnt/{home,opt,boot/efi,var/{cache,lib/libvirt,lib/docker,log,spool,tmp}}

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
mount -vo $BTRFS_OPTS,subvol=@docker  $ROOT_PART /mnt/var/lib/docker
mount -vo $BTRFS_OPTS,subvol=@log     $ROOT_PART /mnt/var/log
mount -vo $BTRFS_OPTS,subvol=@spool   $ROOT_PART /mnt/var/spool
mount -vo $BTRFS_OPTS,subvol=@tmp     $ROOT_PART /mnt/var/tmp
mount -v $EFI_PART /mnt/boot/efi
```

```bash
mount | grep /mnt
btrfs subvolume list /mnt
cat /mnt/root/instalacion.env
```

------

## Fase 3: Bootstrap de Debian

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

```bash
whoami          # debe responder "root"
cat /etc/debian_version
echo "$ROOT_PART / $EFI_PART / $SWAP_PART"   # no debe salir vacío
```

------

## Fase 4: Sistema base

```bash
export HOSTNAME="debian"
```

```bash
# Por si volviste a entrar al chroot en una terminal nueva: restaura PATH y variables
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env

# Se guarda junto al resto de variables por si hace falta recordarlo más adelante
cat >> /root/instalacion.env << EOF
export HOSTNAME="$HOSTNAME"
EOF

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
UUID=$BTRFS_UUID /var/lib/docker  btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@docker 0 0
UUID=$BTRFS_UUID /var/log         btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@log 0 0
UUID=$BTRFS_UUID /var/spool       btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@spool 0 0
UUID=$BTRFS_UUID /var/tmp         btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@tmp 0 0
UUID=$EFI_UUID   /boot/efi        vfat  defaults,noatime 0 2
UUID=$SWAP_UUID  none             swap  defaults 0 0
EOF

echo "$HOSTNAME" > /etc/hostname

cat > /etc/hosts << EOF
127.0.0.1       localhost
127.0.1.1       $HOSTNAME

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

```bash
locale
cat /etc/fstab
dpkg -l | grep linux-image
```

------

## Fase 5: Usuario, hibernación y GRUB

```bash
export FULL_NAME="Diego Salazar"
export USER_NAME="dilusaper"
```

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

```bash
grep resume /etc/default/grub
id $USER_NAME
ls /boot/efi/EFI
```

------

## Fase 6: Redes (Internet)

```bash
# Por si volviste a entrar al chroot en una terminal nueva: restaura PATH y variables
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
source /root/instalacion.env

apt install -y network-manager wpasupplicant wireless-tools rfkill \
               firmware-iwlwifi

systemctl enable NetworkManager
```

```bash
nmcli device status
nmcli device wifi list
sudo nmtui   # conectar a una red desde una interfaz de texto simple
ping -c3 deb.debian.org
```

------

## Fase 7: Primer escritorio gráfico y navegador

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

------

## Fase 8: Audio, pantallas y teclado retroiluminado

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

------

## ✅ HASTA AQUÍ DESDE EL LIVE — sistema mínimo funcional

```bash
exit          # sale del chroot
umount -vR /mnt
reboot
```

------

## Fase 9: Bluetooth, monitores y snapshots (integración de hardware)

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

```bash
bluetoothctl show                 # el adaptador debe aparecer "Powered: yes"
blueman-manager                   # abre la GUI; deberías poder buscar dispositivos
nwg-displays                      # abre la GUI de monitores
dpkg -L gnome-color-manager | grep bin/   # confirma el binario real antes de lanzarlo
btrfs-assistant                   # abre y debe listar tus subvolúmenes @, @home, etc.
```

------

## Fase 10: Aplicaciones de escritorio

```bash
sudo apt install -y kitty nemo
```

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

```bash
kitty --version
nemo                                    # abre el explorador de archivos
flatpak run com.bitwarden.desktop &     # Bitwarden debe abrir con diálogos nativos
synaptic                                # gestor gráfico de paquetes
```

------

## Fase 11: Gestión de sesión (Wlogout)

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

```bash
wlogout   # deben aparecer los íconos de bloquear/suspender/cerrar sesión/reiniciar/apagar
```

------

## Fase 12: Perfiles de energía (TLP)

```bash
sudo apt install -y tlp

# TLP y power-profiles-daemon compiten por el mismo control de energía. Si el
# segundo llegó como dependencia de algo instalado antes, se enmascara para
# que TLP quede como única autoridad.
sudo systemctl mask power-profiles-daemon.service 2>/dev/null || true
sudo systemctl enable tlp.service
```

```bash
mkdir -p ~/.config/power-profiles

cat > ~/.config/power-profiles/bateria.sh << 'EOF'
#!/bin/bash
# Perfil: batería — prioriza autonomía sobre rendimiento
sudo tee /etc/tlp.d/99-perfil.conf > /dev/null << 'CONF'
TLP_DEFAULT_MODE=BAT
CPU_SCALING_GOVERNOR_ON_BAT=powersave
CPU_ENERGY_PERF_POLICY_ON_BAT=power
START_CHARGE_THRESH_BAT0=30
STOP_CHARGE_THRESH_BAT0=90
CONF
sudo tlp start
EOF

cat > ~/.config/power-profiles/intermedio.sh << 'EOF'
#!/bin/bash
# Perfil: intermedio — balance entre autonomía y rendimiento
sudo tee /etc/tlp.d/99-perfil.conf > /dev/null << 'CONF'
TLP_DEFAULT_MODE=AC
CPU_SCALING_GOVERNOR_ON_AC=schedutil
CPU_ENERGY_PERF_POLICY_ON_AC=balance_performance
START_CHARGE_THRESH_BAT0=30
STOP_CHARGE_THRESH_BAT0=90
CONF
sudo tlp start
EOF

cat > ~/.config/power-profiles/rendimiento.sh << 'EOF'
#!/bin/bash
# Perfil: rendimiento — prioriza velocidad, ignora el umbral de carga
sudo tee /etc/tlp.d/99-perfil.conf > /dev/null << 'CONF'
TLP_DEFAULT_MODE=AC
CPU_SCALING_GOVERNOR_ON_AC=performance
CPU_ENERGY_PERF_POLICY_ON_AC=performance
START_CHARGE_THRESH_BAT0=0
STOP_CHARGE_THRESH_BAT0=100
CONF
sudo tlp start
EOF

chmod +x ~/.config/power-profiles/*.sh
```

```bash
sudo tlp-stat -s          # confirma que TLP está activo y qué modo tiene ahora
~/.config/power-profiles/bateria.sh
cat /sys/class/power_supply/BAT0/charge_control_start_threshold  # debe decir 30
cat /sys/class/power_supply/BAT0/charge_control_end_threshold    # debe decir 90
```

```bash
lsblk -p
```

```bash
export DISK="/dev/sdX"
```

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

```bash
update-grub
update-initramfs -u -k all
```

```bash
exit
umount -vR /mnt
reboot
```

------
