![Logo Discord](https://zupimages.net/up/23/26/rumo.png)
[Rejoignez le Discord !](https://discord.gg/rSfTxaW)

[![Utilisateurs en ligne](https://img.shields.io/discord/347412941630341121?style=flat-square&logo=discord&colorB=7289DA)](https://discord.gg/347412941630341121)

### ENVIRONEMENT
```bash
python3 -m venv venv
source venv/bin/activate
pip install ansible
```
```bash
git@github.com:micferna/pxe_tftp.git && pxe_tftp
```
* Modifier le fichier dhcp.yml dans group_var/all/dhcp.yml renseignez les variables puis :
```sh
ansible-playbook -i inventaire.ini main.yml
```
