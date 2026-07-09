# Instalar Debian 13 (trixie) con btrfs + subvolúmenes, desde un Debian Live, para usar con sway

Esta guía reemplaza al instalador gráfico de Debian. En vez de eso, arrancas
un Debian Live (un sistema que corre desde el USB, sin instalar nada) y
construyes el sistema final a mano, comando por comando, con
**debootstrap**. Esto es necesario porque el instalador gráfico de Debian
no permite definir subvolúmenes de btrfs personalizados — solo puedes
elegir "usar btrfs" en general.

Al final de esta guía tendrás un sistema Debian 13 mínimo (sin entorno
gráfico) con btrfs y los subvolúmenes correctos ya separados desde el
origen. Desde ahí, clonas tu repositorio `debian-sway-v2` y corres
`./sway.sh` exactamente como ya tenías planeado — esa parte no cambia.

## Conceptos que necesitas entender antes de empezar

**¿Qué es un subvolumen?** En un sistema de archivos tradicional (ext4,
por ejemplo), si quieres que `/home` esté separado de `/`, necesitas una
partición física distinta, con un tamaño fijo decidido de antemano. En
btrfs, un subvolumen es distinto: es como una carpeta especial dentro del
**mismo** espacio de almacenamiento, que el sistema trata como si fuera
su propio sistema de archivos independiente para efectos de montaje y
snapshots — pero **sin reservarle un tamaño fijo**. Todos los subvolúmenes
dentro de la misma partición btrfs comparten el mismo espacio libre. Por
eso no necesitas decidir "cuánto le doy a cada uno": btrfs lo resuelve
dinámicamente.

**¿Por qué separar `/home`, `/var/log`, etc. en sus propios subvolúmenes?**
Porque Snapper (la herramienta que gestiona snapshots) toma snapshots
**por subvolumen**. Si todo vive dentro de un solo subvolumen `@`, cada
snapshot de tu sistema incluye también tus archivos personales, tus logs,
tu caché, etc. — cosas que no quieres "revertir" cuando haces rollback de
una actualización fallida. Separándolos, un rollback de `@` (el sistema)
no toca lo que vive en los demás subvolúmenes.

**¿Qué subvolúmenes usaremos, y por qué cada uno?**

| Subvolumen | Se monta en | Por qué existe |
|---|---|---|
| `@` | `/` | El sistema operativo en sí: lo único que realmente quieres poder revertir |
| `@home` | `/home` | Tus archivos personales no deben verse afectados si reviertes el sistema |
| `@log` | `/var/log` | Si reviertes `/` tras un fallo, quieres conservar el registro de *qué* falló |
| `@cache` | `/var/cache` | Datos descartables (cachés de apt, fuentes, miniaturas); separarlos aligera cada snapshot de `@` |
| `@tmp` | `/var/tmp` | Igual que `@cache`: temporales que no aportan nada a un snapshot del sistema |
| `@opt` | `/opt` | Software instalado manualmente fuera de apt (no lo confundas con tus `.deb`, esos van a `/usr`) |
| `@spool` | `/var/spool` | Colas de impresión y tareas pendientes del sistema |
| `@libvirt` | `/var/lib/libvirt` | Tus máquinas virtuales (con libvirt/QEMU/KVM) no inflarán cada snapshot de `@` |
| `@respaldos` | `/home/Respaldado` (enlazado como `~/Respaldado`) | Carpeta de propósito general, con la misma estructura de subcarpetas que `home`, donde TÚ decides qué mover según lo que quieras proteger con snapshots (ver explicación abajo) |

**Sobre `@respaldos` — por qué existe y cómo se usa:** dado que tus
archivos personales superan los 200GB regularmente, activar snapshots
sobre todo `@home` resultaría en historiales pesados, porque btrfs
retiene los bloques viejos de cualquier archivo grande que cambie (no
copia los 200GB completos en cada snapshot, pero sí acumula espacio real
con archivos pesados que cambian seguido). En vez de que el sistema
decida qué proteger por tipo de archivo, creamos un subvolumen
**independiente**, montado en una ruta fija (`/home/Respaldado`, que no
depende de ningún usuario en particular) y enlazado dentro de tu home
como `~/Respaldado` para que lo veas como una carpeta normal. Dentro de
ella puedes ir creando las subcarpetas que necesites según qué quieras
proteger (`Respaldado/Documentos`, `Respaldado/Proyectos`, etc.). Solo
este subvolumen tendrá snapshots automáticos — el resto de `/home` queda
protegido de rollbacks de `@` (por estar en `@home`, ya separado), pero
sin historial propio de snapshots, evitando el problema de espacio.

**Importante:** los snapshots de `@respaldos` viven en el mismo disco
físico que todo lo demás. Protegen contra errores de edición, borrados
accidentales o corrupción de un archivo puntual — no contra una falla
del disco completo. Para ese nivel de riesgo, una copia en otro medio
(disco externo, nube, etc.) es la única protección real; eso queda fuera
del alcance de este proyecto y a tu criterio.

No usaremos un subvolumen `@swap`: ya tienes una **partición** de swap
separada (no un archivo dentro de btrfs), que es más simple y funciona
igual de bien para hibernación. La diferencia entre ambos métodos se
explica al final de esta guía, en la sección "Apéndice: swap como
partición vs. swap como archivo".

No usaremos subvolúmenes para gestores de sesión gráficos (`@gdm3`,
`@sddm`, `@lightdm`): esos existen para entornos de escritorio completos
(GNOME, KDE, XFCE). Tú usas **greetd + tuigreet**, que es mucho más
liviano y no tiene un directorio de estado equivalente que necesite
aislarse.

---

## Paso 1 — Arrancar el Debian Live y preparar las herramientas

Arranca tu máquina (virtual o real) desde el **Debian 13 Live ISO**
(no la ISO de instalación normal — necesitas la versión "Live", que es un
sistema completo corriendo desde el USB). Cualquier edición Live sirve
(GNOME, XFCE, etc.); la elección no afecta al sistema final, ya que no
vamos a usar nada de su entorno gráfico, solo su terminal.

Una vez en el entorno Live, abre una terminal y conviértete en root:

```bash
sudo su
```

Identifica el disco de destino:

```bash
lsblk -p
```

Esto lista todos los discos. Busca el que corresponde a tu disco real
(en una VM normalmente aparece como `/dev/sda` o `/dev/vda`). Guárdalo en
una variable para no tener que escribirlo cada vez:

```bash
export DISK="/dev/sda"   # reemplaza por el disco correcto
```

**Advertencia:** todo lo que sigue borra completamente el disco indicado
en `$DISK`. Verifica dos veces que sea el disco correcto antes de seguir.

Instala la herramienta de particionado:

```bash
apt update && apt install -y gdisk
```

---

## Paso 2 — Particionar el disco (EFI + swap + raíz btrfs)

A diferencia del artículo original (que solo crea EFI + raíz, dejando el
swap para después como archivo), aquí creamos **tres particiones desde el
inicio**, porque decidiste mantener el swap como partición clásica:

```bash
# Borra el disco y crea una tabla GPT nueva
sgdisk -Z $DISK
sgdisk -og $DISK

# Partición 1: EFI System Partition (512 MiB)
sgdisk -n 1::+512M -t 1:ef00 -c 1:'EFI' $DISK

# Partición 2: swap (1.5x tu RAM — ajusta el tamaño a tu caso)
# Ejemplo para 8 GB de RAM: 1.5 x 8 = 12 GiB
sgdisk -n 2::+12G -t 2:8200 -c 2:'SWAP' $DISK

# Partición 3: raíz btrfs (todo el espacio restante)
sgdisk -n 3:: -t 3:8300 -c 3:'LINUX' $DISK

# Formatea EFI como FAT32
mkfs.fat -F32 -n EFI ${DISK}1

# Prepara la partición de swap
mkswap -L SWAP ${DISK}2

# Formatea la raíz como btrfs
mkfs.btrfs -L DEBIAN ${DISK}3

# Verifica que todo quedó como esperas
lsblk -po name,size,fstype,fsver,label,uuid $DISK
```

Nota el cambio respecto al artículo original: como ahora la partición
raíz es la **tercera** (no la segunda, porque insertamos el swap en medio),
todas las referencias a `${DISK}2` que verás en guías genéricas pasan a
ser `${DISK}3` en nuestro caso. Ya está ajustado en los comandos de esta
guía — solo es importante que lo notes si comparas con otras fuentes.

---

## Paso 3 — Crear los subvolúmenes de btrfs

```bash
# Monta la raíz btrfs (sin subvol todavía, accedemos al nivel superior)
mount -v ${DISK}3 /mnt

# Crea cada subvolumen según la tabla explicada arriba
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@opt
btrfs subvolume create /mnt/@spool
btrfs subvolume create /mnt/@libvirt
btrfs subvolume create /mnt/@respaldos

# Desmonta; ya cumplió su propósito (crear la estructura)
umount -v /mnt
```

---

## Paso 4 — Montar los subvolúmenes en su disposición final

```bash
# Opciones de montaje: noatime evita escrituras innecesarias al leer
# archivos; space_cache=v2 mejora el rendimiento del propio btrfs;
# compress=zstd:1 comprime de forma transparente y barata en CPU.
BTRFS_OPTS="defaults,noatime,space_cache=v2,compress=zstd:1"

# Monta la raíz (subvolumen @)
mount -vo $BTRFS_OPTS,subvol=@ ${DISK}3 /mnt

# Crea los puntos de montaje para los demás subvolúmenes
mkdir -vp /mnt/{home,opt,boot/efi,var/{cache,lib/libvirt,log,spool,tmp}}

# Monta @home PRIMERO, porque /home/Respaldado debe existir dentro de
# @home ya montado (si creas la carpeta antes de montar @home, la
# carpeta quedaría "perdida" debajo del subvolumen, no dentro de él)
mount -vo $BTRFS_OPTS,subvol=@home ${DISK}3 /mnt/home
mkdir -vp /mnt/home/Respaldado

# Monta el resto de subvolúmenes en su carpeta correspondiente
mount -vo $BTRFS_OPTS,subvol=@opt       ${DISK}3 /mnt/opt
mount -vo $BTRFS_OPTS,subvol=@cache     ${DISK}3 /mnt/var/cache
mount -vo $BTRFS_OPTS,subvol=@libvirt   ${DISK}3 /mnt/var/lib/libvirt
mount -vo $BTRFS_OPTS,subvol=@log       ${DISK}3 /mnt/var/log
mount -vo $BTRFS_OPTS,subvol=@spool     ${DISK}3 /mnt/var/spool
mount -vo $BTRFS_OPTS,subvol=@tmp       ${DISK}3 /mnt/var/tmp
mount -vo $BTRFS_OPTS,subvol=@respaldos ${DISK}3 /mnt/home/Respaldado

# Monta la partición EFI
mount -v ${DISK}1 /mnt/boot/efi

# Verifica el resultado
lsblk -po name,size,fstype,uuid,mountpoints $DISK
```

No montamos el swap aquí: una partición de swap no se "monta" en una
carpeta, se activa más adelante con `swapon` (Paso 9).

---

## Paso 5 — Instalar el sistema base con debootstrap

```bash
apt install -y debootstrap

# Instala el sistema base mínimo de Debian 13 dentro de /mnt
debootstrap --arch=amd64 trixie /mnt http://deb.debian.org/debian

# Monta los sistemas de archivos especiales necesarios para usar chroot
for dir in dev proc sys run; do
    mount -v --rbind "/${dir}" "/mnt/${dir}"
    mount -v --make-rslave "/mnt/${dir}"
done

# Monta las variables EFI (necesario para sistemas UEFI)
mount -v -t efivarfs efivarfs /mnt/sys/firmware/efi/efivars
```

---

## Paso 6 — Configurar `/etc/fstab`

`fstab` es el archivo que le dice al sistema, en cada arranque, qué
montar y dónde. Como tenemos nueve subvolúmenes más la partición EFI y el
swap, cada uno necesita su propia línea:

```bash
BTRFS_UUID=$(blkid -s UUID -o value ${DISK}3) ; echo $BTRFS_UUID
EFI_UUID=$(blkid -s UUID -o value ${DISK}1)   ; echo $EFI_UUID
SWAP_UUID=$(blkid -s UUID -o value ${DISK}2)  ; echo $SWAP_UUID

cat > /mnt/etc/fstab << EOF
UUID=$BTRFS_UUID /                btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@ 0 0
UUID=$BTRFS_UUID /home            btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@home 0 0
UUID=$BTRFS_UUID /opt             btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@opt 0 0
UUID=$BTRFS_UUID /var/cache       btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@cache 0 0
UUID=$BTRFS_UUID /var/lib/libvirt btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@libvirt 0 0
UUID=$BTRFS_UUID /var/log         btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@log 0 0
UUID=$BTRFS_UUID /var/spool       btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@spool 0 0
UUID=$BTRFS_UUID /var/tmp         btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@tmp 0 0
UUID=$BTRFS_UUID /home/Respaldado btrfs defaults,noatime,space_cache=v2,compress=zstd:1,subvol=@respaldos 0 0
UUID=$EFI_UUID   /boot/efi        vfat  defaults,noatime 0 2
UUID=$SWAP_UUID  none             swap  defaults 0 0
EOF

cat /mnt/etc/fstab
```

**Nota sobre el orden de montaje:** no es necesario preocuparte por el
orden exacto de las líneas en `fstab`. Como `/home/Respaldado` está
literalmente dentro de la ruta de `/home`, systemd detecta esa jerarquía
automáticamente y garantiza que `/home` se monte antes que
`/home/Respaldado` en cada arranque, sin necesidad de ninguna opción
adicional.

---

## Paso 7 — Entrar al sistema instalado (chroot)

```bash
chroot /mnt /bin/bash
```

A partir de aquí, todos los comandos se ejecutan **dentro** del sistema
nuevo, como si ya hubieras arrancado en él.

---

## Paso 8 — Configuración básica del sistema

```bash
# Nombre del equipo (cámbialo si quieres otro)
echo "debian" > /etc/hostname

cat > /etc/hosts << EOF
127.0.0.1       localhost
127.0.1.1       $(cat /etc/hostname)

::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
EOF

# Zona horaria — ajusta a tu ubicación real
ln -sf /usr/share/zoneinfo/America/Bogota /etc/localtime

# Configuración regional (idioma del sistema)
apt install -y locales
dpkg-reconfigure locales
```

Cuando `dpkg-reconfigure locales` te muestre la lista, marca la que
corresponda a tu idioma (por ejemplo `es_CO.UTF-8` o `en_US.UTF-8`,
según prefieras) y elige esa misma como predeterminada al final.

---

## Paso 9 — Repositorios, paquetes base, y activar el swap

```bash
cat > /etc/apt/sources.list << EOF
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian trixie main contrib non-free non-free-firmware

deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
deb-src http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
EOF

apt update

# Kernel, firmware, GRUB, red, herramientas básicas
apt install -y linux-image-amd64 linux-headers-amd64 \
    firmware-linux firmware-linux-nonfree \
    grub-efi-amd64 efibootmgr network-manager \
    btrfs-progs sudo vim bash-completion

# Activa el swap definido en fstab
swapon -a
swapon -v
```

---

## Paso 10 — Hibernación con swap en partición

Esta es la parte que cambia respecto al artículo original (que usaba un
archivo de swap dentro de btrfs, con su propio cálculo de "offset"). Con
una **partición** de swap, el procedimiento es más directo: el kernel solo
necesita saber el UUID de la partición, sin offsets adicionales.

```bash
# Verifica que Secure Boot esté deshabilitado (la hibernación lo requiere)
# Si no tienes mokutil instalado todavía, sáltate esta verificación por
# ahora y revísalo desde la BIOS/UEFI de tu máquina directamente.

SWAP_UUID=$(blkid -s UUID -o value ${DISK}2)
echo "GRUB_CMDLINE_LINUX_DEFAULT=\"quiet resume=UUID=$SWAP_UUID\"" >> /etc/default/grub

cat > /etc/initramfs-tools/conf.d/resume << EOF
RESUME=UUID=$SWAP_UUID
EOF

update-initramfs -u -k all
```

(El `update-grub` que aplica esto se ejecuta en el Paso 12, después de
instalar GRUB.)

---

## Paso 11 — Crear tu usuario

```bash
# Reemplaza "tu_usuario" y "Tu Nombre" con tus datos reales
useradd -m -G sudo,adm,libvirt -s /bin/bash -c "Tu Nombre" tu_usuario
passwd tu_usuario
id tu_usuario

# Enlaza la carpeta de respaldos (subvolumen @respaldos, montado en
# /home/Respaldado) dentro del home del usuario, para que la veas como
# ~/Respaldado igual que cualquier otra carpeta normal.
ln -s /home/Respaldado /home/tu_usuario/Respaldado
chown -h tu_usuario:tu_usuario /home/tu_usuario/Respaldado

# Dentro de ~/Respaldado, crea las subcarpetas que vayas necesitando,
# calcando la estructura de tu home, según qué quieras proteger con
# snapshots (ejemplo de inicio, ajusta a tu gusto):
mkdir -p /home/Respaldado/{Documentos,Proyectos}
chown -R tu_usuario:tu_usuario /home/Respaldado
```

Nota: se agregó el grupo `libvirt` al usuario en este paso, ya que
mencionaste que usarás libvirt/QEMU/KVM para tus máquinas virtuales —
sin pertenecer a ese grupo, no podrías gestionarlas sin `sudo` en cada
comando.

```bash
# 1. Crear el grupo libvirt manualmente para que useradd no proteste
groupadd libvirt

# 2. Crear tu usuario (reemplaza "tu_usuario" y "Tu Nombre" con tus datos reales)
useradd -m -G sudo,adm,libvirt -s /bin/bash -c "Tu Nombre" tu_usuario

# 3. Asignar la contraseña a tu nuevo usuario
passwd tu_usuario

# 4. Crear el enlace simbólico hacia el subvolumen de respaldos
ln -s /home/Respaldado /home/tu_usuario/Respaldado
chown -h tu_usuario:tu_usuario /home/tu_usuario/Respaldado

# 5. Crear las subcarpetas internas y ajustar permisos finales de la ruta completa
mkdir -p /home/Respaldado/{Documentos,Proyectos}
chown -R tu_usuario:tu_usuario /home/Respaldado
```

---

## Paso 12 — Instalar y configurar GRUB

```bash
grub-install \
  --target=x86_64-efi \
  --efi-directory=/boot/efi \
  --bootloader-id=debian \
  --recheck

update-grub
```

---

## Paso 13 — Salir y reiniciar

```bash
exit                    # Sale del chroot
umount -vR /mnt         # Desmonta todo
reboot
```

Quita el USB/ISO de arranque cuando reinicie, para que arranque desde el
disco.

---

## A partir de aquí: tu proyecto de sway

Cuando reinicies, llegarás a una consola de texto (sin entorno gráfico,
exactamente como tu premisa original). Inicia sesión con tu usuario y
desde aquí el flujo es el que ya conocías:

```bash
git clone <tu-repositorio>
cd debian-sway-v2
./sway.sh
```

`sway.sh` instalará sway y todo lo demás sobre esta base. No necesita
ningún cambio por haber usado debootstrap en vez del instalador gráfico —
para `sway.sh`, esto ya es "un Debian 13 con btrfs", que es exactamente
lo que esperaba.

La única diferencia es que `scripts/configure_snapper.sh` ahora encuentra
los subvolúmenes `@home`, `@log`, etc. ya separados desde el origen, así
que Snapper puede aprovecharlos correctamente sin que tengas que migrar
nada después (a diferencia del plan de "modo rescue" que habíamos
considerado antes para una instalación ya existente).

---

## Apéndice: swap como partición vs. swap como archivo

Por si te encuentras esta diferencia en otras guías (como el artículo
original que revisamos), aquí está la comparación directa:

| | Partición de swap (lo que usamos aquí) | Archivo de swap dentro de btrfs |
|---|---|---|
| Dónde vive | Su propia partición en la tabla GPT | Un archivo regular dentro de un subvolumen `@swap` |
| Redimensionar | Requiere reparticionar | Se agranda con `dd` + `mkswap`, sin tocar particiones |
| Configuración para hibernar | Solo necesita el UUID de la partición | Necesita además un "offset" físico (`btrfs inspect-internal map-swapfile`), porque btrfs puede fragmentar el archivo en el disco |
| Necesita `chattr +C` / desactivar compresión | No aplica (no es un archivo dentro de btrfs) | Sí, obligatorio, o el rendimiento y la integridad del swap se ven comprometidos |
| Complejidad | Menor | Mayor |

Ambos métodos dan hibernación funcional. Elegimos partición porque es más
simple y porque ya la tenías definida así en tu instalación anterior — no
hay ninguna desventaja real para tu caso de uso por no usar el método de
archivo.





https://chatgpt.com/share/6a500595-f6f4-83e9-9519-540434208f43
