![Logo Discord](https://zupimages.net/up/23/26/rumo.png)
[Rejoignez le Discord !](https://discord.gg/sdNAHEHGzy)

[![Utilisateurs en ligne](https://img.shields.io/discord/347412941630341121?style=flat-square&logo=discord&colorB=7289DA)](https://discord.gg/sdNAHEHGzy)

# pxe_tftp

Playbook Ansible qui déploie un serveur de démarrage réseau sur Debian :

- **DHCP** IPv4/IPv6 via **Kea** (remplace `isc-dhcp-server`, EOL et absent
  de Debian 13)
- **TFTP / PXE** via `tftpd-hpa`, démarrage **BIOS legacy** (pxelinux) **et
  UEFI** (GRUB), avec option **Secure Boot**
- **Installation automatisée** optionnelle par preseed (servie en HTTP)

Compatible Debian 12 (bookworm) et Debian 13 (trixie).

## Rôles

| Rôle        | Rôle joué                                                   |
|-------------|-------------------------------------------------------------|
| `dhcp_role` | Serveur Kea DHCPv4 + DHCPv6, détection d'architecture PXE   |
| `pxe_role`  | Serveur TFTP, bootloaders BIOS/UEFI, images d'installation |
| `http_role` | Serveur web servant les fichiers preseed (si activé)        |

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

## Fonctionnalités optionnelles

### Installation automatisée (preseed)

Désactivée par défaut car elle **efface le disque cible sans confirmation**.
Pour l'activer : `pxe_enable_autoinstall: true` dans `group_vars/all/dhcp.yml`.
Le `http_role` installe alors nginx et publie les fichiers preseed, et des
entrées « [AUTOMATED] » apparaissent dans les menus de démarrage.
Adaptez les réponses preseed dans `http_role/defaults/main.yml`.

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
