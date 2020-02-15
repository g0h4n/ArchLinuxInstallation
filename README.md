# How To Install My ArchLinux?
## Introduction

Dans cette procédure, je vous détaille step by step comment installer votre Arch Linux.

Certaines étapes diffères, pour la connexion Wifi par exemple sur une VM nous sommes naté automatiquement au réseau du poste.
Si nous installons ArchLinux sur un laptop il faudra utiliser le paquet "wifi-menu" ou encore "NetworkManager" avec la commande "nmtui".

Si vous souhaitez installer Arch Linux sur une VM avec VMWARE, veuillez éditer le fichier .vmx afin d'ajouter la ligne suivante à la fin de celui-ci :


```conf
firmware = "efi"
```

## Préparation clé bootable

Afin d'installer le système ArchLinux, il faut télécharger l'ISO de celui-ci depuis le site officiel d'ArchLinux.
http://mir.archlinux.fr/iso/latest/

Une fois votre clé bootable prête, il suffit de booter sur celle-ci et choisir, installer ArchLinux 64bits.


## Installation
### Prépartion

Il faut passer le système sur une version française avec un clavier azerty une fois que vous avez un shell qui apparaît.

```bash
loadkeys fr
```

Maintenant que nous avons notre clavier azerty, nous allons vérifier que nous sommes bien connecté à internet.

```bash
ping google.fr
```

Si la requête ICMP passe nous pouvons passer à l'étape suivante, sinon vérifier votre connexion réseau.
Pour le paramétrage wifi utiliser le paquet 'wifi-menu'.


### Partitionnement

Nous allons partitionner notre système de façon à avoir les partion suivantes :

```tree
\boot\  1Go
\swap\  4Go
\       30Go
\home\  15Go
```

Pour le partitionnement vérifier le nom de votre disque avec la commande suivante :

```bash
fdisk -l
```

Ici mon disque c'est /dev/sda


Création des partions :

```bash
gdisk /dev/nvme0n1
    d    	#delete all partitions
    n    	# Add a new partition
    1    	# partition number (/boot)
    2048 	# On commence à 2048
    +1000M 	# Et je finis 100Mo plus loin

    t
    ef00    # EFI system


    # On créée les suivantes

    n       # Nouvelle partition
    2       # Partition numéro 2 (swap)
    [enter] # par défaut on accepte le secteur proposé
    +16G     # Je crée une partition de 4Go pour le swap
    t       # Nous allons changer le type de patition.
    2       # On choisit la patition 2
    8200      # On définit la partition comme swap


    n       # Nouvelle partition
    3       # partition numéro 3 (/)
    [enter]
    +49G    # 


    n       # Nouvelle partition
    4       # partition numéro 4 (/home)
    [enter]        
    [enter]

    w       # On sauve les changements
```


Formatage des partitions  :


```bash
mkfs.fat -F32 /dev/sda1
mkswap /dev/sda2
mkfs.ext4 /dev/sda3
mkfs.ext4 /dev/sda4
```

Monter la partition /dev/sda3 

```bash
mount /dev/sda3 /mnt/
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
swapon /dev/sda2
```


### Installation du système de base

Sélection du miroir :


```bash
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
#mettre les depot France au début du fichier
vim /etc/pacman.d/mirrorlist
```


Installation des paquets de base :

```bash
pacstrap /mnt base git zsh firefox curl base-devel intel-ucode dialog wpa_supplicant vim 
```


### Configuration hostname et languages

La commande ci-dessous permet de passer sur notre ArchLinux :

```bash
arch-chroot /mnt
```

Configuration du hostname de la machine :

```bash
echo 'G0H4N-VM-ARCHLINUX' > /etc/hostname
```

Configuration de la langue et des fuseaux horaires :

```bash
#Décommenter les deux lignes ci-dessous :

vim /etc/locale.gen
    fr_FR.UTF-8 UTF-8
    fr_FR ISO-8859-1

locale-gen

echo LANG=fr_FR.UTF-8 > /etc/locale.conf

export LANG=fr_FR.UTF-8

echo KEYMAP=fr > /etc/vconsole.conf

ln -s /usr/share/zoneinfo/Europe/Paris /etc/localtime
```


### Configuration du boot

Afin que notre système puisse démarrer correctement nous allons préparer notre /boot :

```bash
pacman -S linux
bootctl --path=/boot install
```

```bash
vim /boot/loader/loader.conf

    default search
    timeout 3
    editor no
    default ID
```


Rrécupérer le PARTUUID de la partion racine / avec la commande suivante :

```bash
blkid /dev/sda3
```


Ajouter ce PARTUUID dans le fichier suivant :

```bash
vim /boot/loader/entries/arch.conf

    title Arch Linux
    linux /vmlinuz-linux
    initrd /intel-ucode.img
    initrd /initramfs-linux.img
    options root=PARTUUID="66cda2d8-4aca-49ad-b1ff-ac17dd70cb01" rw quiet
```


Enfin mettre à jour notre /boot :

```bash
bootctl --path=/boot update
```

Redémarrer votre poste ou votre VM.


## Configuration
### Après redémarrage


Vérifier la connexion internet.

Mettre à jour son sytème.


```bash
pacman -Syu
```


Mettre un mot de passe root à son ArchLinux :

```bash
passwd
```


### Chiffrement du /home

Nous allons chiffrer la partion /home/ avec le paquet cryptsetup.
Mais avant cela nous devons la umount pour la libèrer.

```bash
umount /home
```

```bash
cryptsetup -y -v luksFormat /dev/sda4

    Are you sure? : YES
```


Ouvrez maintenant votre nouvelle partition chiffrée et créez un système de fichiers pour l’utiliser. 
Ici j'ia choisi ext4, mais vous pouvez choisir ce que vous voulez.

En ouvrant la partition, la passphrase saisie précédemment vous sera demandée.


L’étape dd n’est pas nécessaire, mais elle permet une protection supplémentaire pour protéger vos données.
dd écrit des 0 sur la partition, cela peut prendre du temps suivant la taille de la partition.

```bash
#Ouverture de la partition chiffré.
cryptsetup luksOpen /dev/sda4 home

#la commande dd est longue et non obligatoire.
dd if=/dev/zero of=/dev/mapper/home

#Formatage de la partition chiffré.
mkfs.ext4 /dev/mapper/home
```


Votre partition est désormais prête à l’emploi. 
Si vous voulez chiffrer votre clé USB/disque externe avec LUKS, vous pouvez utiliser la même procédure.


Pour utiliser la nouvelle partition chiffrée avec /home, vous devrez faire quelques chanegements à la fois sur fstab et sur crypttab pour la monter correctement.
Éditez le fichier /mnt/etc/crypttab en ajoutant la ligne suivante. 

Si vous ne voulez pas de temps d’attente, supprimez cette option.


```bash
home /dev/sda4 none luks,timeout=120
```


Veuillez noter que tous les changements faits sur la partition (par exemple avec le gestionnaire de partitions) nécessitent que la partition soit débloquée sinon, son chiffrement sera effacé.


```bash
cryptsetup luksOpen /dev/sda4 home
e2label /dev/mapper/home <name>
```


Nous pouvons maintenant terminer de monter la partion :

```bash
mkdir /mnt/home && mount /dev/mapper/home /mnt/home
```


Recherchez dans le fichier /mnt/etc/fstab la ligne de montage /home, et remplacez UUID par /dev/mapper/home. 



### Compte personnel


Création d'un compte utilisateur :

```bash
$ useradd -mg users -s /bin/zsh g0h4n
$ pacman -S sudo
$ export EDITOR=vim
$ visudo

    g0h4n ALL=(ALL) ALL

$ reboot
```


### Environement graphique (i3)


Installer les paquets suivants :

```bash
$ pacman -S i3 dmenu xorg xorg-xinit lightdm
```


Ajouter la ligne suivante afin de démarrer i3 au démarrage session.

```bash
$ vim ~/.xinitrc to this:

	#! /bin/bash
	exec i3
```


Logez-vous appuyer sur 'ENTRER' et choisir la touche de command.
Quitter et redémarrer afin d'avoir un shell.


Editer le fichier de config d'i3 :

```bash
$ vim ~/.config/i3/config
```



Installer poplybar :

```bash
$ yay polybar
```


Personnaliser la bar de status.


```bash
$ vim ~/.i3status
```


Installation d'un terminal avec i3 :

```bash
$ pacman -S deepin-terminal
```


Nous voulons qu'i3 se lance tout seul (startx) :

```bash
$ vim /etc/profile

	# autostart systemd default session on tty1
	if [[ "$(tty)" == '/dev/tty1' ]]; then
	    exec startx
	fi
```


## Personnalisation de son Arch

Maintenant, je vais installer un autre gestionnaire de paquet 'yay' :


```bash
$ sudo pacman -S git firefox
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
$ yay
```


Voila, maintenant pour installer un paquet il faut utiliser la commande 'yay <nomdupaquet>' et pour mettre à jour ceux déjà installer juste un 'yay' suffit.
Pour supprimer un paquet déjà installer 'yay -R <nomdupaquet>'.


## Installation des dépots blackarch (like kali linux)

Récupèrer le script d'installation sur blackarch.org : 

```bash
$ curl -O https://blackarch.org/strap.sh
```


Le sha1sum du script récupéré doit matcher avec celui-ci : 9f770789df3b7803105e5fbc19212889674cd503 strap.sh

```bash
$ sha1sum strap.sh
```


Nous rendons le script éxécutable et nous pouvons le lancer :

```bash
$ chmod +x strap.sh
$ sudo ./strap.sh
$ yay
```


Installer le nouveau shell 

```bash
$ yay zsh neofetch
```


Changer la variable environement du shell : 

```bash
$ export SHELL=/bin/zsh
$ chsh -s /bin/zsh
```


## Personnaliser son shell

Pimp son shell zsh avec le paquet 'ohmyzsh'.
(Cela s'applique à l'utilisateur qui execute la commande)

```bash
$ sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```


Il faut ensuite changer le style en éditant ~/.zshrc et remplacer le style ligne 11 par agnoster :

```bash
ZSH_THEME="agnoster"
```


Editer le fichier ~/.zshrc et ajouter à la fin de celui-ci :

```bash
clear
neofetch

#Alias
alias ll="ls -lisa --color=auto"
alias shutdown="shutdown -F now"
```


Désactiver l'historique dans le zsh_history avec un lien symbolique (question de sécurité), ce lien symbolique fonctionne sur tout type de fichier :
Pareil en root :

```bash
$ ln -sf /dev/null ~/.zsh_history
```


## Personnaliser l'éditeur 'vim'

Editer le ~/.vimrc et coller ceci : 

```bash
set nu
set noshowmode
set ai
set expandtab
set tabstop=4
set shiftwidth=4
set nocompatible
set background=dark
syntax on
set wildmenu
set wildmode=list:longest,list:full
set listchars=eol:¬,tab:>—,trail:♦,nbsp:⍽,extends:>,precedes:<
set list
set foldmethod=syntax
set hidden
set termguicolors
command W :execute ':silent w !sudo tee % > /dev/null | :edit!'
let g:gruvbox_italic=1
let g:airline_powerline_fonts=1
let g:airline#extensions#tabline#enabled=1
let g:airline#extensions#vimtex#enabled=1
let g:airline#extensions#syntastic#enabled=1
let g:airline#extensions#tagbar#enabled=1
let g:airline#extensions#tmuxline#enabled=1
let g:airline#extensions#languageclient#enabled=1
``` 


Editer yay pacman et ajouter les deux lignes suivantes : 

```bash
$ vim /ect/pacman.conf
    Color 
    ILoveCandy
```


Notre Arch Linux est prêt pour l'utilisation.
