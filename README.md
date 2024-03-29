# Instalação do Slackware15 em sistema de arquivos BTRFS.

Simples tutorial demonstrando como instalar o Slackware 15.0  em sistema de arquivos BTRFS com subvolumes.  
A instalaçao segue estilo "Arch Linux", via linha de comando..

## 1 - Iniciar com o dvd/usb de instalação do Slackware. 

Para facilitar, eu vou fazer a instalação via ssh. Depois do boot e configuração do teclado, vamos definir outras configurações:

- definir senha:
```
passwd
```

- definir IP via dhcp:
```
dhcpcd eth0
```

- iniciar o serviço de ssh server:
```
/etc/rc.d/rc.dropbear start
```
<br>

## 2 - Formatação:

- Conectar via ssh para fazer a instalação remotamente:
```
putty root@ip_do_slackware
```

Como demonstração, vou configurar as partições em um HD modo "GPT" de 20GB :  

| Mount point | Partition | Partition type | FS Type     | Bootable flag | Size   |
|-----------|-----------|---------------------|-----------|----|--------|
| /boot/efi | /dev/sda1 | EFI System Partition| FAT32 | Yes           | 512 MiB|
| SWAP      | /dev/sda2 | Linux swap | SWAP   | No            | 2 GiB |
| /         | /dev/sda3 | Linux | BTRFS       | No            | 17,5 GiB |

- Para facilitar a instalação vamos criar a variável "disk" apontando para o HD de instalação:
```
disk='/dev/sda'
```

- Formatar as partições:
```
mkfs.fat -F32 -n EFI ${disk}1
mkswap -L SWAP ${disk}2; swapon ${disk}2
mkfs.btrfs -f -L ROOT ${disk}3
```
<br>

## 3 - Criação do volume principal e subvolumes:
```
mount -t btrfs ${disk}3 /mnt

btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@opt
btrfs subvolume create /mnt/@srv
btrfs subvolume create /mnt/@tmp
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@.snapshots

umount /mnt
```
<br>

## 4 - Remontagem dos volumes:

- Montagem dos volumes:
```
mount -o noatime,space_cache=v2,subvol=@ ${disk}3 /mnt

mkdir /mnt/{boot,home,opt,srv,tmp,var,.snapshots}
mount -o noatime,space_cache=v2,subvol=@home ${disk}3 /mnt/home
mount -o noatime,space_cache=v2,subvol=@opt ${disk}3 /mnt/opt
mount -o noatime,space_cache=v2,subvol=@srv ${disk}3 /mnt/srv
mount -o noatime,space_cache=v2,subvol=@tmp ${disk}3 /mnt/tmp
mount -o noatime,space_cache=v2,subvol=@var ${disk}3 /mnt/var
mount -o noatime,space_cache=v2,subvol=@.snapshots ${disk}3 /mnt/.snapshots
```

- Partição EFI:
```
mkdir /mnt/boot/efi
mount ${disk}1 /mnt/boot/efi -t vfat
mkdir -p /mnt/boot/efi/EFI
```
<br>

## 5 - Instalação dos pacotes:

Como demonstração, estou usando as séries "a ap d l n". Sem interface gráfica.
```
mount /dev/cdrom /cdrom
for pkg in a ap d l n; do installpkg --terse --root /mnt /cdrom/slackware64/$pkg/*.txz; done
```
<br>

## 6 - Criação do arquivo fstab:
```
cat <<EOF > /mnt/etc/fstab
#fstab:
      
${disk}1    /boot/efi       vfat    defaults                                                           1    0
${disk}2    swap            swap    defaults                                                           0    0
${disk}3    /               btrfs   noatime,space_cache=v2,subvol=/@,subvol=@                          0    0
${disk}3    /home           btrfs   noatime,space_cache=v2,subvol=/@home,subvol=@home                  0    0
${disk}3    /opt            btrfs   noatime,space_cache=v2,subvol=/@opt,subvol=@opt                    0    0
${disk}3    /srv            btrfs   noatime,space_cache=v2,subvol=/@srv,subvol=@srv                    0    0
${disk}3    /tmp            btrfs   noatime,space_cache=v2,subvol=/@tmp,subvol=@tmp                    0    0
${disk}3    /var            btrfs   noatime,space_cache=v2,subvol=/@var,subvol=@var                    0    0
${disk}3    /.snapshots     btrfs   noatime,space_cache=v2,subvol=/@.snapshots,subvol=@.snapshots      0    0
devpts      /dev/pts        devpts  gid=5,mode=620                                                     0    0
proc        /proc           proc    defaults                                                           0    0
tmpfs       /dev/shm        tmpfs   nosuid,nodev,noexec                                                0    0
EOF
```
<br>  

## 7 - Instalação EFI:
```
mount -t proc /proc /mnt/proc
mount -o bind /dev /mnt/dev
mount -o bind /sys /mnt/sys

source /mnt/etc/profile
chroot /mnt /usr/sbin/grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub
chroot /mnt /usr/sbin/grub-mkconfig -o /boot/grub/grub.cfg
```
<br>

## 8 - Ajustes:

- Definir o teclado para br-abnt2:
```
cat <<EOF > /mnt/etc/rc.d/rc.keymap
#!/bin/sh
# Load the keyboard map. More maps are in /usr/share/kbd/keymaps.

if [ -x /usr/bin/loadkeys ]; then
  /usr/bin/loadkeys br-abnt2.map
fi
EOF

chmod +x /mnt/etc/rc.d/rc.keymap
```

- Definir senha para usuário "root":
```
chroot /mnt /usr/bin/passwd
```

- Definir as configurações da placa de rede eth0:
```
cp -L /etc/resolv.conf /mnt/etc/resolv.conf
chroot /mnt /sbin/netconfig
```

- Acertar horário:
```
chroot /mnt /usr/sbin/timeconfig  
```

- Atualizar certificados CA:
```
chroot /mnt /usr/sbin/update-ca-certificates --fresh 1  
```
<br>

## 9 - Finalização:

- Desmontar volumes e reiniciar:
```
umount /mnt/sys
umount /mnt/dev
umount /mnt/proc

reboot
```

Espero ter ajudado!
