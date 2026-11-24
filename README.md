# odios-arch

Portage **Arch / Manjaro** de [odios](https://github.com/b0bbywan/odios), l'installeur
Ansible de streamer audio (AirPlay, Spotify/Qobuz Connect, MPD, UPnP, Bluetooth,
multiroom Snapcast…) conçu à l'origine pour **Debian/Ubuntu/Raspbian** (apt/`.deb`).

> odios est publié sous licence **BSD-2-Clause**, © 2026 Mathieu Réquillart
> (`b0bbywan`). Ce dépôt est un fork qui conserve cette licence et l'attribution.
> Base du portage : odios **2026.11.0rc1**.

## Principe

On garde intégralement le playbook Ansible d'odios ; on remplace **uniquement** la
couche paquets `apt`/`.deb` par :

- `community.general.pacman` pour les paquets des dépôts officiels Arch ;
- un **dépôt pacman local `[odios]`** qui sert 7 paquets « maison » reconstruits
  depuis les sources GitHub d'odios via des `PKGBUILD` (dans `packages/`), plus
  2 paquets AUR (`snapcast`, `python-snapcast`).

Les `.deb` d'odios installent dans des chemins standards identiques sur Arch
(`/usr/bin`, `/usr/lib/systemd/user/`, `/usr/share/<pkg>/…`) : les `PKGBUILD`
reproduisent la même arborescence, donc les rôles Ansible ne changent que leur
tâche d'installation de paquet — templates, activation systemd et édition de
config restent inchangés.

Détails complets du portage (mapping Debian→Arch, correctifs runtime) :
voir [`PORTING.md`](PORTING.md).

## Prérequis

```bash
sudo pacman -S --needed ansible base-devel git
ansible-galaxy collection install community.general
```

## Installation

1. **Construire les paquets maison et le dépôt local `[odios]`** (depuis `packages/`) :

   ```bash
   mkdir -p repo
   for p in packages/*/; do (cd "$p" && makepkg -sf --noconfirm); done
   cp packages/*/*.pkg.tar.zst repo/
   repo-add repo/odios.db.tar.zst repo/*.pkg.tar.zst
   ```

   (Les 2 paquets AUR `snapcast` / `python-snapcast` se construisent de la même
   façon ou via un helper AUR.)

2. **Déclarer le dépôt** dans `/etc/pacman.conf` (le rôle `common` l'ajoute aussi
   automatiquement) :

   ```ini
   [odios]
   SigLevel = Optional TrustAll
   Server = file:///chemin/vers/odios-arch/repo
   ```

   ```bash
   sudo pacman -Sy
   ```

3. **Lancer le playbook** avec tes variables (copie et édite l'exemple) :

   ```bash
   cp extra-vars.example.json extra-vars.json   # édite target_user, target_hostname, etc.
   cd ansible
   ansible-playbook -i inventory/localhost.yml playbook.yml -e @../extra-vars.json
   ```

## État

Portage **installé et validé** sur Manjaro x86_64 (backend **pipewire**), services
utilisateur actifs, 0 échec. Fonctions testées de bout en bout :

- ✅ **AirPlay** (shairport-sync + nqptp)
- ✅ **Qobuz Connect** (qbzd)
- ✅ **Spotify Connect** (spotifyd)
- ✅ **Bluetooth** A2DP (PipeWire)
- ✅ **UPnP/DLNA** (upmpdcli), **MPD** + myMPD, **Snapcast**, **odio-api** (UI web :8018)

## Hors périmètre / connu

- Pas de bootstrap `install.sh` Arch (installation via la commande `ansible-playbook`).
- L'auto-upgrade d'odioctl ré-exécute l'installeur Debian : non adapté au dépôt
  `[odios]` (timer désactivé).

## Licence

BSD-2-Clause — voir [`LICENSE`](LICENSE). Code original odios © Mathieu Réquillart.
