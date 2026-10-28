# Buildroot sur QEMU — x86_64 et ARM64

Projet personnel d'apprentissage : construire des images Linux embarqué depuis
les sources avec Buildroot, sans matériel, et comprendre chaque brique.

## Environnement

| Élément | Version |
|---|---|
| Hôte | Ubuntu 24.04.4 LTS (VM VMware, 8 cœurs, 8 Go RAM) |
| Buildroot | 2026.02.3 (LTS) |
| Noyau cible | Linux 6.18.7 |
| Toolchain | GCC 14.3.0 + glibc, construite par Buildroot |
| Émulateur | QEMU 8.2.2 |

## Ce qui a été fait

1. Image **x86_64** à partir de `qemu_x86_64_defconfig`, démarrée sous QEMU
2. Personnalisation via `menuconfig` : hostname, mot de passe root haché en
   SHA-512, ajout de `htop` (dépendance `ncurses` résolue par Kconfig)
3. Image **ARM64 (Cortex-A53)** en compilation croisée à partir de
   `qemu_aarch64_virt_defconfig`, dans un dossier de sortie séparé (`O=`)

## Observations

- **Toolchain** : Buildroot construit son propre GCC 14.3.0, distinct de celui
  de l'hôte (13.3.0), avec un sysroot isolé.
- **Démarrage** : noyau → montage du rootfs en lecture seule → `/sbin/init`
  (BusyBox) → `/etc/inittab` → scripts `/etc/init.d/S??*` → `getty`.
- **BusyBox** : un binaire d'environ 750 Ko fournissant 279 commandes par liens
  symboliques. Rootfs complet de 5,5 Mo.
- **Recompilation** : après ajout de `htop`, 1 min 30 au lieu d'environ 1 h,
  car seuls 5 paquets manquants sont construits (fichiers `.stamp_*`).
- **Compilation croisée** : le binaire ARM est refusé par l'hôte
  (`Exec format error`), `file` indique `ARM aarch64`.
- **Device Tree (ARM64)** : le noyau découvre le matériel par le Device Tree
  (UART `arm,pl011` à `0x09000000`, contrôleur d'interruptions GIC),
  là où le x86_64 utilise ACPI et PCI.

## Reproduire

    git clone https://gitlab.com/buildroot.org/buildroot.git
    cd buildroot && git checkout 2026.02.3
    make BR2_DEFCONFIG=/chemin/vers/configs/bene_x86_64_defconfig defconfig
    make
    output/images/start-qemu.sh serial-only

## Limites

- Validation sous QEMU uniquement, sans matériel réel.
- Configurations de carte fournies par Buildroot, adaptées mais non écrites
  de zéro.
- Mot de passe root retiré du defconfig publié.
