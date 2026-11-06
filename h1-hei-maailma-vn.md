# h1 Hei maailma! ja muita perusasioita

*Ville Nietula, Haaga-Helian tunnus configurations management - kurssin kotitehtävä h1. Työ tehty 6.11.2026.*

## Tiivistelmä (x)

### Karvinen 2026: SSH public key - Login without password

- Käy läpi ssh-paketin käytön: kirjautumisen ssh:llä, avainparin luonnin (`ssh-keygen`) ja julkisen avaimen siirtämisen palvelimelle (`ssh-copy-id`), minkä jälkeen kirjautuminen onnistuu ilman salasanaa
- Keskeinen periaate: julkinen avain menee orjan `~/.ssh/authorized_keys`-tiedostoon, yksityinen avain pysyy koneella
- **Oma huomio:** sama ketju (keygen → copy-id → salasanoton kirjautuminen) toimi tässä tehtävässä sellaisenaan, ja samaa avainparia voi hyödyntää myös GitHubissa

### Karvinen 2026: Hello Ansible

- Käy Ansiblen käytön läpi selkeästi: asennuksen, inventaarion, playbooken ajon ja yksinkertaisten tehtävien automatisoinnin
- Purkaa myös Ansiblen kansiorakenteen ja roolit, eli miten isompi projekti kannattaa järjestää
- **Oma huomio:** artikkelia voi käyttää tehtävän pohjana; roolien ja kansiorakenteen kuvaus oli hyötyä, kun järjestin oman projektini rakenteen

## a) Virtuaalikone ja alkutoimet

Työskentely-ympäristönä oli kurssin Debian 13 -orjakone "orja", erillinen Hyper-V-virtuaalikone, joka on luotu samalla menetelmällä kuin h0-raportin kone (hypervisori, ISO-asennus ja alkutoimet on dokumentoitu tarkemmin [h0 Oikeasti netissä](https://github.com/veenetul/VNlab-Docs/blob/main/h0-oikeasti-netissa-vn.md) -raportissa). Isäntäkoneena AMD Ryzen 5 4500U ja 32 Gt RAM-muistilla varustettu läppäri, isäntäkäyttöjärjestelmä Windows 11, virtualisointina Hyper-V. Tässä raportissa keskityn Ansible-alkutoihin: tarkistin ip-osoitteen, päivitin paketit ja asensin ssh-palvelimen ja Ansiblen:

```bash
hostname -I          # → 172.27.103.9
sudo apt-get update
sudo apt-get upgrade
sudo apt install openssh-server ansible -y
sudo systemctl enable --now ssh
```

Ssh-palvelin lähtee käyntiin automaattisesti bootissa (`enabled`) ja oli `active (running)`.

apt-get update/upgrade ja openssh-server + ansiblen asennussystemctl status ssh: active (running)

## Versionhallinta

Asensin gitin ja tein globaalit käyttäjäasetukset:

```bash
sudo apt install git -y

git config --global user.name "veenetul"
git config --global user.email "ville.nietula@example.com"
```

Globaalit asetukset tallentuvat `~/.gitconfig`-tiedostoon, joten ne pätevät kaikkiin repositorioihin ilman toistoa.

Lisäsin julkisen avaimen GitHubin asetuksiin (SSH keys), joten kloonaus onnistuu ilman salasanaa.

GitHubin SSH-avaimet

Ssh-palvelimen tila varmistettiin systemctlillä (kuva kohdassa a): `active (running)`, `enabled`.

Koska olin tehnyt kurssin aikana GitHubiin oman repositorion, jossa oli valmiiksi Ansible-kansiorakenne, konfiguraatio (`ansible.cfg`) ja inventaariotiedosto, sain ne kaikki kloonaamalla:

```bash
git clone git@github.com:…/VNlab-Docs.git
# → toimiva rakenne: ansible.cfg, inventory, site.yml ...
```

Näin vältin rakenteen ja asetustiedostojen tekemisen uudelleen, ja kotitehtävät syntyvät suoraan versiohallintaan.

## b) SSH-yhteys

Openssh-palvelin oli asennettuna alkutoimissa. Yhdistin ensin salasanalla Windows 11:n PowerShellistä isäntäkoneelta:

```powershell
ssh ladmin@172.27.103.9        # salasanakirjautuminen
```

Sitten tein avainparin ja otin käyttöön avainpohjaisen kirjautumisen:

```bash
ssh-keygen                     # avainpari (ed25519)
ssh-copy-id ladmin@172.27.103.9   # "Number of key(s) added: 1"
ssh ladmin@172.27.103.9        # ei salasanakyselyä
```

Julkinen avain on lisätty myös GitHubiin, joten kloonaus ilman salasanaa onnistuu.

ssh-yhteys PowerShellistä: ladmin@orjassh-keygen: ed25519-avainparissh-copy-id: "Number of key(s) added: 1"

## c) Ansible ja inventaario

Loin projektikansioon inventaarion ja asetustiedoston:

```bash
cd ~/VNlab-Docs/ansible
nano ansible.cfg    # inventory = ./inventory
nano inventory
```

Inventaario on repositoriossa: [ansible/inventory](https://github.com/veenetul/VNlab-Docs/blob/main/ansible/inventory). Se määrittelee ryhmän `orjat`, johon orjakone kuuluu (`ansible_connection=ssh`, `ansible_user=ladmin`).

Testi:

```bash
ansible localhost -m ping
# localhost | SUCCESS => { "ping": "pong" }
```

Asetustiedosto `ansible.cfg` - kiinnitin myös Python-tulkinnan, jolloin tulkkivaroitus poistuu eikä tulkki vaihdu järjestelmäpäivityksessä:

```ini
[defaults]
inventory = ./inventory
interpreter_python = /usr/bin/python3.13
```

inventory nano-editorissaansible localhost -m ping → pongansible.cfg: inventory + interpreter\_python

## d) Hei maailma! -playbook

Koko playbook on repositoriossa: [ansible/h1.yml](https://github.com/veenetul/VNlab-Docs/blob/main/ansible/h1.yml). Se sisältää viisi tehtävää:

1. `ping` - yhteystesti
2. `file` - kansion `/tmp/toimiiko` luonti (mode 0755)
3. `copy` - tiedoston `heimaailma` kirjoitus (mode 0644)
4. `debug` - viesti "Hei maailma!"

Ajo ja tulos:

```bash
ansible-playbook h1.yml -i inventory
```

- `testaa yhteys`: ok
- `luo kansio /tmp/toimiiko`: changed
- `tiedostonkirjoitus`: changed
- `hei maailma`: ok - tulostaa `"msg": "Hei maailma!"`
- PLAY RECAP: `ok=5 changed=2 failed=0`

Toinen ajo: `ok=5 changed=0` - idempotenssi kunnossa, mikään ei muuttunut.

Tiedoston sisältö todennettu:

```bash
cat /tmp/toimiiko/heimaailma
# → testi
```

site.yml nano-editorissaEnsimmäinen ajo: ok=5 changed=2, Python-tulkki-varoitus näkyyToinen ajo: ok=5 changed=0 - idempotenssiheimaailma-tiedoston sisältö "testi"/tmp/toimiiko olemassa, heimaailma listattuna

## Vianetsintä

Ensimmäisellä yrityksellä sain virheitä, jotka ratkaisin yksi kerrallaan:

1. **"no hosts matched"** - ajoin playbookin `-i hosts` -lipulla, mutta inventaariotiedostoni nimi on `inventory`. Korjaus: `ansible-playbook h1.yml -i inventory` (tai ansible.cfg, jolloin lippua ei tarvita).
2. **YAML-virhe** - tehtävissä oli kaksoispiste, jonka jälkeen ei tullut välilyöntiä. YAML vaatii `avain: arvo` -muodon, korjasin sisennyksen ja välilyönnit.
3. **Python-tulkki-varoitus** - \[WARNING\] Python-tulkista (/usr/bin/python3.13). Tämä on vain varoitus, ei virhe; ajot menivät läpi. Kiinnitin tulkin `ansible.cfg`-tiedostoon (`interpreter_python = /usr/bin/python3.13`), jolloin varoitus poistui ja tulkki ei vaihdu järjestelmäpäivityksessä.

Virhetilanne: no hosts matched ja YAML-virhe

## Maininta tekoälyn käytöstä

Markdown-tiedostojen luomisessa ja editoinnissa käytin apuna Mistral AI:n Vibe-tekoälyä ([https://mistral.ai](https://mistral.ai/)). Kaikki tekemiset, komennot ja valinnat raportissa ovat omia, ja kuvakaappaukset otin itse; tekoäly toimi dokumentoinnin apuvälineenä.

## Lopputulos

Ensimmäinen Ansible-playbook toimii ja on idempotentti. Muutokset työnnetty GitHubiin:

```bash
git add --all && git commit -m "h1 & h2: reports, screenshots, playbooks" && git pull && git push
```

git add, commit ja push

Repositoriossa tehtävän playbook on tiedostossa `ansible/h1.yml` (h2-tehtävän playbook on `site.yml`/`h2.yml`), joten molemmat ovat suoraan ajettavissa ja todennettavissa.

## Lähteet

- Karvinen 2025: [Configuration Management 2025 autumn](https://terokarvinen.com/configuration-management/) - tehtävänanto
- Karvinen 2026: [SSH public key - Login without password](https://terokarvinen.com/ssh-public-key-login-without-password/) - x) tiivistelmään
- Karvinen 2026: [Hello Ansible](https://terokarvinen.com/hello-ansible/) - x) tiivistelmään
- Karvinen 2006: [Hello World!](https://terokarvinen.com/2006/hello-world/) - raportointikielestä