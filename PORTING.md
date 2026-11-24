# odios-arch — portage Manjaro/Arch d'odios

Fork d'odios (upstream `b0bbywan/odios`, base `2026.11.0rc1`) pour Arch/Manjaro.
On garde le playbook Ansible ; on remplace la couche paquets `apt`/`.deb` par
`pacman` + AUR + PKGBUILD maison.

## Principe clé
Les `.deb` maison installent dans des chemins **standards et identiques sur Arch** :
`/usr/bin`, `/usr/lib/systemd/user/`, `/usr/share/<pkg>/config.yaml`,
`/etc/sudoers.d/`, `/usr/share/dbus-1/…`. Nos PKGBUILD reproduisent la MÊME
arborescence → les rôles Ansible ne changent que leur tâche d'installation de
paquet, tout le reste (templates, activation systemd, édition de config) est
inchangé.

## Mapping des paquets

### Standards (dépôts Arch)
| Debian | Arch | source |
|---|---|---|
| pulseaudio / pulseaudio-module-zeroconf | pulseaudio / pulseaudio-zeroconf | extra |
| pipewire / pipewire-pulse / wireplumber | pipewire / pipewire-pulse / wireplumber | extra (déjà là) |
| bluez / rfkill | bluez / bluez-utils / util-linux | extra (déjà là) |
| mpd / mpc | mpd / mpc | extra |
| python3-pip | python-pip | extra |
| python3-musicbrainzngs | python-musicbrainzngs | extra |
| acl | acl | core (déjà là) |
| polkitd | polkit | extra |
| unattended-upgrades | — (N/A : pas d'apt ; voir rôle upgrade) | — |
| pi-bluetooth | — (N/A : Raspberry Pi only) | — |

### AUR (PKGBUILD à récupérer ou écrire)
| Debian | AUR | note |
|---|---|---|
| shairport-sync | shairport-sync | AirPlay 2 possible |
| snapclient | snapcast | fournit snapclient |
| spotifyd | spotifyd | Rust |
| upmpdcli | upmpdcli | + libupnpp/libnpupnp |
| mympd | mympd | web UI MPD |

### Maison → PKGBUILD depuis GitHub (reproduire layout .deb)
| paquet | version .deb | langage | repo | fichiers clés |
|---|---|---|---|---|
| odio-api | 0.18.2 | Go | go-odio-api | /usr/bin/odio-api, user unit odio-api.service, /usr/share/odio-api/config.yaml |
| odioctl | 0.4.2 | Go | odioctl | /usr/bin/odioctl, 6 user units (odioctl-web, odioctl-web-proxy.socket, odio-check-upgrade.{service,timer}, odio-upgrade.service, odioctl-web.socket), /etc/sudoers.d/odioctl ; postinst crée groupes system `odio`+`odioctl`, /var/lib/odio (2770 root:odio), /var/cache/odio (2775) |
| mpd-discplayer | 0.9.0 | Go | go-mpd-discplayer | /usr/bin, user unit, /usr/share/mpd-discplayer/config.yaml ; deps runtime: libcdio-paranoia, libdiscid, libgudev |
| qbzd | 2.1.2 | Rust | qbz (odio-qbz?) | /usr/bin/qbzd, user unit qbzd.service, complétions bash/zsh/fish ; deps: alsa-lib, openssl |
| snapclientmpris | 1.3.1 | Python | snapclientmpris | /usr/bin, user+system unit, dbus service+conf, /usr/share/snapclientmpris/snapclientmpris.conf ; deps py: python-snapcast, python-dbus-fast, python-zeroconf |
| mpd2mpris | 0.13.1 | Python | mpd2mpris | /usr/bin, user unit, dbus service, /usr/share/doc/.../mpd2mpris.conf ; deps py: python-mpd(2), python-dbus-fast, python-musicbrainzngs |
| odio-kiosk | 2.0.1 | C++/Qt6 | odio-framebuffer-ui | /usr/bin/odio-kiosk, user unit odio-kiosk@.service, /etc/default/odio-kiosk ; deps: qt6-webengine, qt6-base, nss |

## Chantiers
- [x] 0. Toolchain : rust installé (base-devel/qt6-webengine déjà présents). + FIX : toolchain Go cassé (14482 fichiers orphelins dans /usr/lib/go) → wipe+réinstall go.
- [x] 1. Rôle `common` : clé/source apt retirées → ajoute le dépôt pacman `[odios]` à pacman.conf + `pacman -Sy`.
- [x] 2. Couche paquets : les 27 `ansible.builtin.apt` → `community.general.pacman` ; install_recommends & contraintes `>=` retirées.
- [x] 3. PKGBUILD maison (7) buildés : odio-api, odioctl, mpd-discplayer, qbzd(-bin), snapclientmpris, mpd2mpris, odio-kiosk. Dans fork/packages/.
- [x] 4. AUR : seuls snapcast + python-snapcast (reste = dépôts officiels Manjaro). Dépôt local [odios] monté (fork/../repo).
- [x] 5. Réécriture des 16 rôles : noms Arch, pi-bluetooth/rfkill retirés, plugins upmpdcli (fournis par le paquet), display arch (uname), polkit.
- [x] 6. Rôle `upgrade` : unattended-upgrades neutralisé (Arch = pacman).
- [x] 7. Install + test bout-en-bout OK (backend pipewire, user pib) : playbook failed=0, 10 services actifs, odio-api :8018/ui HTTP 200, myMPD :8080 OK.

## Correctifs runtime rencontrés (différences Debian→Arch / état machine)
- Toolchain Go : /usr/lib/go avait 14482 fichiers orphelins (ancienne version) → wipe + réinstall `go`.
- libnfs : système en upgrade partiel (6.0.2 installé, mpd veut libnfs.so.17) → `pacman -S libnfs` (7.0.2).
- shairport-sync : backend `pa` (Debian) → `pipewire` (natif dans le build Arch). Var corrigée dans le rôle.
- shairport-sync (AirPlay 2) : exige le daemon PTP `nqptp` en service SYSTÈME. Le paquet Arch `shairport-sync` en dépend (nqptp présent) mais n'active jamais le service → crash-loop « can not find the nqptp service ». Rôle corrigé : install nqptp + `systemctl enable --now nqptp.service`.
- qbzd : retrait de `conflicts=('qbz')` pour coexister avec qbz-bin (appli desktop déjà installée).
- extra-vars : target_hostname ne doit pas être vide (sert au device_name qbzd).
- odio-check-upgrade.timer : désactivé (MAJ auto propre à l'installeur Debian, hors périmètre).
- odio-api UI sans style : /ui/static/output.css manquant (CSS Tailwind non compilé). PKGBUILD odio-api corrigé pour compiler le CSS (tailwindcss v3.4.17) avant go build → embarqué.
- pipewire / snapcast-discover : le rôle `pipewire` déploie un drop-in chargeant `libpipewire-module-snapcast-discover` en OBLIGATOIRE ; ce module n'existe pas dans le PipeWire d'Arch → pipewire échoue au démarrage (`start-limit-hit`, status 254) et TOUT l'audio tombe (AirPlay/BT/Qobuz muets), typiquement après un reboot. Template `snapcast-discover.conf.j2` corrigé : `flags = [ ifexists ]` (chargé seulement si présent).

## Note session systemd user
Le gestionnaire systemd --user de pib doit être relancé (re-login/reboot) pour
hériter des groupes odio/odioctl/bluetooth ajoutés — sinon les services lisant
/var/lib/odio/state.json peuvent échouer tant que la session n'est pas renouvelée.

## Emplacements
- Fork Ansible : /home/pib/odios-arch/fork/ansible  (playbook.yml, roles/, group_vars/)
- PKGBUILD maison : /home/pib/odios-arch/fork/packages/
- Dépôt pacman local [odios] : /home/pib/odios-arch/repo  (enregistré dans /etc/pacman.conf)
- Sources amont clonées : /home/pib/odios-arch/src ; .deb de référence : /home/pib/odios-arch/ref
- Variables d'install : /home/pib/odios-arch/fork/extra-vars.json
- Relancer : cd fork/ansible && ansible-playbook -i inventory/localhost.yml playbook.yml -e @../extra-vars.json

## Mapping noms de paquets (Debian → Arch) appliqué
- pulseaudio-module-zeroconf → pulseaudio-zeroconf ; pulseaudio-module-bluetooth → pulseaudio-bluetooth
- bluez + rfkill → bluez + bluez-utils (rfkill ∈ util-linux) ; pi-bluetooth → supprimé (RPi only)
- snapclient (paquet) → snapcast ; polkitd → polkit
- python3-pip → python-pip ; python3-musicbrainzngs → python-musicbrainzngs
- upmpdcli-{qobuz,tidal,radios,…} → inclus dans le paquet upmpdcli (plugins non séparés)
- libspa-0.2-bluetooth → néant (PipeWire BT intégré)

## Reste à faire (hors périmètre install initiale)
- install.sh : écrire un bootstrap Arch (pacman -S ansible + build dépôt [odios]) en remplacement du script Debian.
- odioctl auto-upgrade : son chemin de MAJ ré-exécute install.sh (Debian) ; à adapter au dépôt [odios] (non bloquant).

## Décisions
- Backend audio : **pipewire** (déjà installé ; évite conflit PulseAudio).
- Machine cible : Manjaro x86_64 (ce poste).
- Install « au fur et à mesure » autorisée ; confirmer avant chaque mutation système.
