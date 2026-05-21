![Logo Discord](https://zupimages.net/up/23/26/rumo.png)
[Rejoignez le Discord !](https://discord.gg/sdNAHEHGzy)

[![Utilisateurs en ligne](https://img.shields.io/discord/347412941630341121?style=flat-square&logo=discord&colorB=7289DA)](https://discord.gg/sdNAHEHGzy)

# pxe_tftp

Playbook Ansible qui déploie un serveur de démarrage réseau sur Debian :

- **DHCP** IPv4/IPv6 via **Kea** (remplace `isc-dhcp-server`, EOL et absent
  de Debian 13)
- **TFTP / PXE** via `tftpd-hpa`, démarrage **BIOS legacy** (pxelinux) **et
  UEFI** (GRUB), avec option **Secure Boot**
- **Provisioning de flotte déclaratif** : chaque machine connue est épinglée
  à une IP et un profil de boot (réservation Kea + config PXE par MAC)
- **Menu universel** : chainload de netboot.xyz (catalogue d'OS et d'outils)
- **Installation automatisée** optionnelle par preseed (servie en HTTP)
- **UEFI HTTP Boot** optionnel et **tableau de bord** Kea (API + page web)

Compatible Debian 12 (bookworm) et Debian 13 (trixie).

## Rôles

| Rôle        | Rôle joué                                                   |
|-------------|-------------------------------------------------------------|
| `dhcp_role` | Serveur Kea DHCPv4 + DHCPv6, réservations de flotte, API   |
| `pxe_role`  | Serveur TFTP, bootloaders BIOS/UEFI, images, menus par MAC |
| `http_role` | Serveur web : preseed, HTTP Boot, page de statut (si activé)|

## Prérequis

- Une machine Debian 12/13 cible, accessible en SSH
- Ansible 2.12+ sur la machine de contrôle

## Environnement

```bash
python3 -m venv venv
source venv/bin/activate
pip install ansible
```

```bash
git clone git@github.com:micferna/pxe_tftp.git
cd pxe_tftp
```

## Configuration

1. **`inventaire.ini`** : adresse et utilisateur SSH du serveur. Privilégiez
   un utilisateur non-root avec `sudo` ; le playbook élève les privilèges
   lui-même (`become: true`).

2. **`group_vars/all/dhcp.yml`** : réseau (plages DHCP, DNS, passerelle,
   `pxe_server_ip`, bootfiles).

3. Réglages techniques optionnels dans `*/defaults/main.yml`
   (versions des installeurs, options TFTP, Secure Boot...).

## Déploiement

```sh
ansible-playbook -i inventaire.ini main.yml
```

## Tests

Validation statique (lancée aussi en CI) :

```sh
yamllint .
ansible-lint
```

Test d'intégration avec **Molecule** : exécute le playbook dans des conteneurs
Debian 12 et 13 et vérifie l'idempotence, la validité des configs Kea et le
démarrage des services. Le démarrage PXE réseau lui-même reste à valider sur
du matériel client.

```sh
pip install ansible "molecule-plugins[docker]" molecule
molecule test
```

## Fonctionnalités optionnelles

### Provisioning de flotte déclaratif

Décrivez vos machines connues dans `pxe_fleet` (`group_vars/all/dhcp.yml`) :

```yaml
pxe_fleet:
  - name: web01
    mac: "aa:bb:cc:dd:ee:01"
    ip: "192.168.1.50"
    os: debian
    autoinstall: true
```

Chaque entrée génère une **réservation Kea** (IP fixe par MAC) et une **config
PXE dédiée** (`pxelinux.cfg/01-<mac>` et `grub/host-<mac>.cfg`) : la machine
démarre directement sur le profil qui lui est assigné. Avec `autoinstall: true`
elle s'installe sans intervention (nécessite `pxe_enable_autoinstall: true`).

### Menu de boot universel (netboot.xyz)

Activé par défaut (`pxe_enable_netbootxyz`). Une entrée de menu chainload
**netboot.xyz**, qui donne accès à un large catalogue d'installeurs d'OS et
d'outils de diagnostic/rescue (memtest, SystemRescue, Clonezilla...).

### Installation automatisée (preseed)

Désactivée par défaut car elle **efface le disque cible sans confirmation**.
Pour l'activer : `pxe_enable_autoinstall: true` dans `group_vars/all/dhcp.yml`.
Le `http_role` installe alors nginx et publie les fichiers preseed, et des
entrées « [AUTOMATED] » apparaissent dans les menus de démarrage.
Adaptez les réponses preseed dans `http_role/defaults/main.yml`.

### UEFI HTTP Boot

`pxe_enable_http_boot: true` : les firmwares UEFI compatibles HTTP Boot se
voient servir une URL `http://` (au lieu du TFTP). Une image GRUB dédiée est
construite et le serveur web publie l'arbre de boot.
**À valider sur du matériel réel** : le comportement dépend du firmware.

### Tableau de bord Kea

`pxe_enable_dashboard: true` : active le **Kea Control Agent** (API REST, sur
la loopback) et génère une page de statut (`http://<serveur>/status.html`)
listant les baux DHCP actifs, rafraîchie par un timer systemd.

### UEFI Secure Boot

Définissez `pxe_secureboot: true` (`pxe_role/defaults/main.yml`) et
`pxe_bootfile_uefi: "bootx64.efi"` (`group_vars/all/dhcp.yml`). Le shim et le
GRUB signés par Debian sont alors publiés via TFTP.
**À valider sur du matériel réel** : le comportement dépend du firmware.

## Sécurité

- **Téléchargements en HTTPS** : noyaux et initrd Debian/Ubuntu récupérés
  exclusivement en HTTPS (empêche l'injection d'un initrd malveillant).
- **Vérification d'intégrité (recommandée)** : renseignez le champ `checksum`
  de `pxe_installer_files` (`pxe_role/defaults/main.yml`), format
  `sha256:<empreinte>`. Un champ vide désactive la vérification.
- **Validation de configuration** : les configs Kea sont validées
  (`kea-dhcp4 -t`) avant déploiement ; un redémarrage de service échoué fait
  désormais échouer le playbook.
- **TFTP en lecture seule** : `tftpd-hpa` tourne avec `--secure` (chroot) et
  sans `--create`.
- **`authoritative`** : activé par défaut (`dhcp_authoritative`). Ne le
  laissez actif que si ce serveur est le **seul** DHCP du segment réseau.
- **Preseed** : les fichiers preseed sont servis en HTTP non authentifié.
  N'y placez pas de secret réel, restreignez le serveur web au réseau de
  provisioning, et chiffrez les identifiants via Ansible Vault.

## Limites connues

- Le PXE/TFTP reste un protocole sans authentification ni chiffrement ;
  Secure Boot fournit un démarrage vérifié côté firmware. Pour un transport
  chiffré, une alternative est iPXE avec amorçage HTTPS.
- Les images netboot d'Ubuntu ne sont fournies que pour les versions
  « legacy » (Focal 20.04) ; adaptez les URLs selon vos besoins.
