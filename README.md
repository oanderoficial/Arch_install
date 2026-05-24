# Guia Completo — Arch Linux + KDE Plasma + Btrfs + LVM + Snapper 

## Objetivo

Instalar um Arch Linux moderno com:

* GPT + UEFI
* EFI montada em `/efi`
* `/boot` dentro do Btrfs
* LVM
* Btrfs
* Root separado do Home
* Subvolumes
* KDE Plasma
* GRUB
* Snapper
* grub-btrfs
* snapshots automáticos
* rollback consistente
* PipeWire
* Flatpak
* SSD otimizado
* visual KDE moderno

---

# Estrutura Final 

```text
EFI FAT32 -> /efi
vgarch-root -> /
vgarch-home -> /home
/boot -> dentro do Btrfs
```

---

# Com esta estrutura:

* snapshots incluem kernel/initramfs
* rollback mais fácl
* `/home` fica separado
* snapshots ficam menores
* root não enche rapidamente
* recovery fica muito mais simples

---

# 1. Boot no pendrive Arch Linux

Escolher:

```text
Arch Linux install medium (x86_64, UEFI)
```

Confirmar UEFI:

```bash
ls /sys/firmware/efi/efivars
```

---

# 2. Ajustar teclado ABNT2

```bash
loadkeys br-abnt2
```

---

# 3. Configurar internet

## Opção A — Cabo de rede

Verificar interfaces:

```bash
ip link
```

Ativar interface:

```bash
ip link set enp3s0 up
```

Obter IP:

```bash
dhcpcd enp3s0
```

Testar:

```bash
ping archlinux.org
```

## Opção B — Wi-Fi / Wireless

Se preferir conectar por wireless, use o `iwctl` no ambiente live do Arch.

Entrar no modo interativo:

```bash
iwctl
```

Listar dispositivos Wi-Fi:

```text
device list
```

Substitua `wlan0` pelo nome mostrado em `device list`.

Procurar redes disponíveis:

```text
station wlan0 scan
```

Listar redes encontradas:

```text
station wlan0 get-networks
```

Conectar na rede:

```text
station wlan0 connect "NOME_DA_REDE"
```

Digite a senha quando solicitado.

Sair do `iwctl`:

```text
exit
```

Testar conexão:

```bash
ping archlinux.org
```

Se não obter IP automaticamente, tente:

```bash
dhcpcd wlan0
```

---

# 4. Corrigir DNS e IPv6 temporariamente

```bash
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.default.disable_ipv6=1
```

```bash
rm -f /etc/resolv.conf
cat > /etc/resolv.conf << EOF
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF
```

---

# 5. Limpar SSD

⚠️ APAGA TUDO.

```bash
sgdisk --zap-all /dev/sda
wipefs -af /dev/sda
partprobe /dev/sda
```

---

# 6. Criar GPT

```bash
parted /dev/sda mklabel gpt
```

---

# 7. Criar EFI

```bash
parted /dev/sda mkpart ESP fat32 1MiB 513MiB
parted /dev/sda set 1 esp on
```

---

# 8. Criar LVM

```bash
parted /dev/sda mkpart primary 513MiB 100%
partprobe /dev/sda
```

---

# 9. Formatar EFI

```bash
mkfs.fat -F32 /dev/sda1
```

---

# 10. Criar LVM

## Criar PV

```bash
pvcreate /dev/sda2
```

## Criar VG

```bash
vgcreate vgarch /dev/sda2
```

## Criar swap

```bash
lvcreate -L 8G vgarch -n swap
```

## Criar root separado

```bash
lvcreate -L 80G vgarch -n root
```

## Criar home separado

```bash
lvcreate -l 100%FREE vgarch -n home
```

## Ativar volumes

```bash
vgchange -ay
```

---

# 11. Formatar Btrfs e swap

## Root

```bash
mkfs.btrfs -L ArchRoot /dev/vgarch/root
```

## Home separado

```bash
mkfs.btrfs -L ArchHome /dev/vgarch/home
```

## Swap

```bash
mkswap -L ArchSwap /dev/vgarch/swap
swapon /dev/vgarch/swap
```

---

# 12. Criar subvolumes Btrfs

## Root

```bash
mount /dev/vgarch/root /mnt
```

Criar subvolumes:

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@cache
btrfs subvolume create /mnt/@log
```

Desmontar:

```bash
umount /mnt
```

---

## Home separado

```bash
mount /dev/vgarch/home /mnt
```

Criar subvolume home:

```bash
btrfs subvolume create /mnt/@home
```

Desmontar:

```bash
umount /mnt
```

---

# 13. Montar corretamente

## Root

```bash
mount -o rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@ /dev/vgarch/root /mnt
```

---

## Home separado

```bash
mkdir -p /mnt/home
mount -o rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@home /dev/vgarch/home /mnt/home
```

---

## Snapshots

```bash
mkdir -p /mnt/.snapshots
mount -o rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@snapshots /dev/vgarch/root /mnt/.snapshots
```

---

## Cache

```bash
mkdir -p /mnt/var/cache
mount -o rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@cache /dev/vgarch/root /mnt/var/cache
```

---

## Logs

```bash
mkdir -p /mnt/var/log
mount -o rw,relatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@log /dev/vgarch/root /mnt/var/log
```

---

## EFI

⚠️ IMPORTANTE:

EFI será montada em `/efi`, NÃO em `/boot`.

```bash
mkdir -p /mnt/efi
mount /dev/sda1 /mnt/efi
```

---

## Boot

`/boot` será apenas uma pasta normal dentro do Btrfs.

```bash
mkdir -p /mnt/boot
```

⚠️ NÃO montar nada em `/mnt/boot`.

---

# 14. Instalar base Arch

```bash
pacstrap -K /mnt base base-devel linux linux-firmware \
btrfs-progs lvm2 sudo nano vim networkmanager \
grub efibootmgr git wget curl \
snapper grub-btrfs snap-pac inotify-tools
```

---

# 15. Gerar fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Verificar:

```bash
cat /mnt/etc/fstab
```

⚠️ O `/home` deve apontar para o UUID do `vgarch-home`.

⚠️ NÃO deve existir:

```fstab
UUID=XXXX /boot vfat
```

Correto:

```fstab
UUID=XXXX /efi vfat
```

---

# 16. Entrar no sistema

```bash
arch-chroot /mnt
```

---

# 17. Timezone

```bash
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
hwclock --systohc
```

---

# 18. Locale

```bash
nano /etc/locale.gen
```

Descomentar:

```text
en_US.UTF-8 UTF-8
pt_BR.UTF-8 UTF-8
```

Gerar:

```bash
locale-gen
```

Idioma:

```bash
echo "LANG=pt_BR.UTF-8" > /etc/locale.conf
```

Teclado:

```bash
echo "KEYMAP=br-abnt2" > /etc/vconsole.conf
```

---

# 19. Hostname

```bash
echo "archlinux" > /etc/hostname
```

---

# 20. Senha root

```bash
passwd
```

---

# 21. mkinitcpio

```bash
nano /etc/mkinitcpio.conf
```

HOOKS:

```text
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block lvm2 filesystems fsck)
```

Gerar initramfs:

```bash
mkinitcpio -P
```

---

# 22. Instalar GRUB EFI

```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
```

Gerar config:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

---

# 23. Validar estrutura correta

## EFI

```bash
findmnt /efi
```

Esperado:

```text
/efi -> vfat
```

---

## Boot

```bash
findmnt /boot
```

Esperado:

```text
SEM saída
```

Isso é CORRETO.

---

## Verificar boot files

```bash
ls /boot
```

Esperado:

```text
grub
initramfs-linux.img
vmlinuz-linux
```

---

## Verificar EFI

```bash
ls /efi
```

Esperado:

```text
EFI
```

⚠️ NÃO deve existir:

```text
vmlinuz-linux
initramfs-linux.img
```

---

# 24. Ativar NetworkManager

```bash
systemctl enable NetworkManager
```

---

# 25. Criar usuário

```bash
useradd -m -G wheel -s /bin/bash anderson
passwd anderson
```

---

# 26. Liberar sudo

```bash
EDITOR=nano visudo
```

Descomentar:

```text
%wheel ALL=(ALL:ALL) ALL
```

---

# 27. Instalar KDE Plasma

Mais enxuto:

```bash
pacman -S plasma-desktop plasma-workspace sddm dolphin konsole firefox
```

Mais completo:

```bash
pacman -S plasma kde-applications sddm dolphin konsole firefox
```

Ativar SDDM:

```bash
systemctl enable sddm
```

---

# 28. Reiniciar

```bash
exit
umount -R /mnt
reboot
```

---

# 29. Configurar Snapper corretamente

## Desmontar snapshots temporariamente

```bash
sudo umount /.snapshots
sudo rm -rf /.snapshots
```

---

## Criar config Snapper

```bash
sudo snapper -c root create-config /
```

---

## Remover subvolume temporário

```bash
sudo btrfs subvolume delete /.snapshots
```

---

## Recriar montagem correta

```bash
sudo mkdir /.snapshots
sudo mount -a
```

Verificar:

```bash
findmnt /.snapshots
```

---

# 30. Ativar timers Snapper

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

---

# 31. Ativar grub-btrfs

```bash
sudo systemctl enable --now grub-btrfsd.service
```

Atualizar GRUB:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

# 32. Snapshot estável

```bash
sudo snapper create --description "Sistema-estavel"
```

---

# 33. Otimização SSD

```bash
sudo systemctl enable --now fstrim.timer
```

---

# 34. PipeWire

```bash
sudo pacman -S pipewire pipewire-pulse pipewire-alsa pipewire-jack wireplumber pavucontrol
```

---

# 35. Flatpak

```bash
sudo pacman -S flatpak
```

```bash
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

---

# 36. Visual KDE moderno

```bash
sudo pacman -S kvantum qt5ct qt6ct papirus-icon-theme
```

---

# 37. Blur e transparência KDE

Ativar em:

```text
Configurações do Sistema
→ Efeitos da Área de Trabalho
```

Ativar:

* Blur
* Transparência
* Fade
* Sliding Popups

---

# 38. NVIDIA GTX

## Opção estável temporária

```bash
sudo pacman -S mesa vulkan-swrast lib32-mesa
```

---

# 39. Recovery rápido

Boot pelo pendrive Arch.

Ativar LVM:

```bash
vgchange -ay
```

Montar root:

```bash
mount -o subvol=@ /dev/vgarch/root /mnt
```

Montar home:

```bash
mkdir -p /mnt/home
mount -o subvol=@home /dev/vgarch/home /mnt/home
```

Montar snapshots:

```bash
mkdir -p /mnt/.snapshots
mount -o subvol=@snapshots /dev/vgarch/root /mnt/.snapshots
```

Montar EFI:

```bash
mkdir -p /mnt/efi
mount /dev/sda1 /mnt/efi
```

Entrar:

```bash
arch-chroot /mnt
```

Reinstalar kernel:

```bash
pacman -S linux linux-firmware mkinitcpio
mkinitcpio -P
```

Reinstalar GRUB:

```bash
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

Reiniciar:

```bash
exit
umount -R /mnt
reboot
```

---

# Resultado Final

Sistema final:

* Arch Linux
* KDE Plasma
* EFI em `/efi`
* `/boot` dentro do Btrfs
* LVM
* Btrfs
* root separado do home
* Snapper
* snap-pac
* grub-btrfs
* snapshots automáticos
* rollback consistente
* recovery simplificado
* kernel sincronizado com snapshots
