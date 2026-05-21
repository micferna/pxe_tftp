![Logo Discord](https://zupimages.net/up/23/26/rumo.png)
[Rejoignez le Discord !](https://discord.gg/rSfTxaW)

[![Utilisateurs en ligne](https://img.shields.io/discord/347412941630341121?style=flat-square&logo=discord&colorB=7289DA)](https://discord.gg/347412941630341121)

# pxe_tftp

Playbook Ansible qui déploie un serveur de démarrage réseau sur Debian :

- **DHCP** (IPv4 et IPv6) via `isc-dhcp-server`
- **TFTP / PXE** via `tftpd-hpa`, avec prise en charge du démarrage
  **BIOS legacy** (pxelinux) **et UEFI** (GRUB)
- Téléchargement automatique des images d'installation Debian et Ubuntu

## Prérequis

- Une machine Debian 12 (bookworm) cible, accessible en SSH
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

1. **`inventaire.ini`** : renseignez l'adresse et l'utilisateur SSH du serveur.
   Privilégiez un utilisateur non-root disposant de `sudo` ; le playbook
   élève les privilèges lui-même (`become: true`).

2. **`group_vars/all/dhcp.yml`** : renseignez le réseau (plage DHCP, DNS,
   passerelle, `pxe_server_ip`, etc.).

3. Réglages techniques optionnels dans `dhcp_role/defaults/main.yml` et
   `pxe_role/defaults/main.yml` (versions des installeurs, options TFTP,
   activation de l'UEFI...).

## Déploiement

```sh
ansible-playbook -i inventaire.ini main.yml
```

## Sécurité

- **Téléchargements en HTTPS** : les noyaux et initrd Debian/Ubuntu sont
  récupérés exclusivement en HTTPS, ce qui empêche l'injection d'un initrd
  malveillant par un attaquant réseau (MITM).
- **Vérification d'intégrité (recommandée)** : renseignez le champ `checksum`
  de `pxe_installer_files` dans `pxe_role/defaults/main.yml`
  (format `sha256:<empreinte>`). Les empreintes officielles se trouvent dans
  les fichiers `SHA256SUMS` publiés à côté des images. Un champ vide désactive
  la vérification.
- **Validation de configuration** : `dhcpd.conf` / `dhcpd6.conf` sont validés
  (`dhcpd -t`) avant déploiement ; un redémarrage de service échoué fait
  désormais échouer le playbook au lieu d'être ignoré silencieusement.
- **TFTP en lecture seule** : `tftpd-hpa` tourne avec `--secure` (chroot) et
  sans `--create`.
- **`authoritative;`** : activé par défaut (`dhcp_authoritative`). Ne le
  laissez actif que si ce serveur est le **seul** DHCP du segment réseau.

## Limites connues

- Le PXE/TFTP reste un protocole sans authentification ni chiffrement ; pour
  un démarrage vérifié, envisagez UEFI Secure Boot ou iPXE en HTTPS.
- `isc-dhcp-server` est en fin de vie en amont et absent de Debian 13
  (trixie) ; une migration vers **Kea** sera nécessaire à terme.
- Les images netboot d'Ubuntu ne sont fournies que pour les versions
  « legacy » (Focal 20.04) ; adaptez les URLs selon vos besoins.
