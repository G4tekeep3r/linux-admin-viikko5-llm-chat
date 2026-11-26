# LLM-chat cPouta (CIC/CD-harjoitus, viikko 5)

Web-chat, jossa käyttäjä keskustelee palvelimella paikallisesti pyörivän kielimallin kanssa. Sovellus asennetaan olemassa olevalle cPouta-palvelimelle (palvelin A) osoitteeseen `/viikko5/`, ja jokainen push `main`-haaraan testataan ja julkaistaan automaattisesti GitHub Actionsilla.

## Arkkitehtuuri

```
Selain --HTTP:80--> Nginx (olemassa oleva sivusto)
                      |-- /viikko2/, /viikko3/, ...   aiemmat viikot, ennallaan
                      '-- /viikko5/  (snippet)
                            |-- /viikko5/static/  -> tiedostot suoraan levyltä
                            '-- unix-soketti --> Gunicorn + Flask --HTTP--> Ollama (127.0.0.1:11434)
                                                      |
                                                   MariaDB (tietokanta chatapp)

GitHub Actions --SSH:22--> rsync + scripts/deploy.sh
```

- Nginx on ainoa palvelu, johon pääsee internetistä (portit 80 ja 22).
- Gunicorn kuuntelee Unix-sokettia `/run/chatapp/gunicorn.sock`, joten se ei törmää muiden sovellusten portteihin.
- Ollama kuuntelee vain osoitetta `127.0.0.1:11434`.
- Keskusteluhistoria tallennetaan MariaDB:n tauluun `messages`.

## Repositorion rakenne

```
.
├── app.py                      # Flask: /, /health, /api/chat (striimaus), /api/history/<id>
├── db.py                       # MariaDB-kerros (muistissa ilman DB_NAME-asetusta)
├── test_app.py                 # pytest-testit
├── templates/index.html        # Chat-näkymä
├── static/app.js, style.css    # Frontend
├── requirements.txt            # Ajonaikaiset riippuvuudet
├── requirements-dev.txt        # + pytest, ruff
├── pyproject.toml              # ruff- ja pytest-asetukset
├── deploy/
│   ├── nginx/chatapp.conf      # Nginx-snippet (location /viikko5/)
│   ├── systemd/chatapp.service # Gunicorn systemd-palveluna
│   └── sql/schema.sql          # Tietokantaskeema
├── scripts/
│   ├── server-setup.sh         # Kertaluonteinen palvelimen alustus
│   └── deploy.sh               # Actionsin ajama julkaisuskripti
└── .github/workflows/ci-cd.yml # CI/CD-putki
```
## Ajo paikallisesti

Vaatii Python 3.12:n. Testit eivät tarvitse Ollamaa eikä tietokantaa.

```powershell
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements-dev.txt
ruff check .
pytest
```

## Palvelimen asennus (kerran)

Kohde on olemassa oleva Ubuntu-palvelin cPoutassa. Muut sivut ja palvelut jätetään ennalleen.

1. Kopioi ja aja alustusskripti palvelimella (käyttäjänä `ubuntu`):
   ```bash
   scp scripts/server-setup.sh ubuntu@<floating-ip>:~
   ssh ubuntu@<floating-ip>
   bash server-setup.sh qwen2.5:0.5b
   ```
   Skripti tarkistaa resurssit ja käytössä olevat portit, asentaa puuttuvat paketit, luo järjestelmäkäyttäjän `chatapp`, MariaDB-tietokannan ja -käyttäjän, kirjoittaa salaisuudet tiedostoon `/etc/chatapp/chatapp.env` (`root:chatapp`, `640`), asentaa Ollaman ja lataa mallin. Tietokannan salasana arvotaan palvelimella, eikä se ole Gitissä.

2. Liitä Nginx-snippet olemassa olevaan sivustoon kerran. Lisää `server { }`-lohkoon rivi:
   ```nginx
   include snippets/chatapp.conf;
   ```
   Testaa ja ota käyttöön:
   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```

## Deploy-avain ja GitHub Secrets

GitHub Actions käyttää omaa SSH-avainparia (ei henkilökohtaista avainta):

```powershell
ssh-keygen -t ed25519 -f $HOME\.ssh\cpouta_deploy -C "github-actions-deploy"
```

Julkinen avain lisätään palvelimen `~/.ssh/authorized_keys`-tiedostoon. Yksityinen avain ja palvelimen tunniste tallennetaan GitHubiin (Settings → Secrets and variables → Actions):

| Secret | Sisältö |
|---|---|
| `CPOUTA_HOST` | palvelimen floating IP |
| `CPOUTA_USER` | `ubuntu` |
| `CPOUTA_SSH_KEY` | deploy-avaimen yksityinen avain |
| `CPOUTA_KNOWN_HOSTS` | palvelimen host key (`ssh-keygen -F <ip>`) |

Avaimia, `.env`-tiedostoja tai salasanoja ei viedä repositorioon (ks. `.gitignore`).

## CI/CD-putki

Workflow: `.github/workflows/ci-cd.yml`

| Tapahtuma | Mitä ajetaan |
|---|---|
| Pull request | Lint ja testit: `ruff check .`, `pytest`, `bash -n scripts/*.sh` |
| Push `main`-haaraan | Lint ja testit, ja vain jos ne menevät läpi, deploy |

Deploy-vaihe:

1. Asettaa SSH-avaimen ja known_hosts-tiedoston secreteistä.
2. Kopioi koodin palvelimelle (`rsync`) hakemistoon `/opt/chatapp/app`.
3. Ajaa palvelimella `scripts/deploy.sh <commit-SHA>`:
   - päivittää virtuaaliympäristön ja riippuvuudet
   - ajaa tietokantaskeeman (`CREATE TABLE IF NOT EXISTS`)
   - päivittää Nginx-snippetin, testaa sen (`nginx -t`) ja lataa Nginxin uudelleen; virheessä palautetaan edellinen snippet
   - päivittää systemd-palvelun ja käynnistää uudelleen vain palvelun `chatapp`
   - varmistaa, että käynnissä oleva versio vastaa deployattua commitia
4. Julkinen savutesti: `/viikko5/health` palauttaa saman commit-SHA:n kuin `github.sha`, ja `/viikko5/` sisältää chat-lomakkeen.

Jos testi epäonnistuu, deploy-jobia ei ajeta. Jos versio ei käynnisty, skripti tulostaa palvelun lokin ja ajo näkyy punaisena.

## Käyttöoikeudet ja salaisuudet

| Polku palvelimella | Sisältö |
|---|---|
| `/opt/chatapp/app` | Sovelluksen koodi (rsync Actionsista) |
| `/opt/chatapp/venv` | Python-virtuaaliympäristö |
| `/etc/chatapp/chatapp.env` | Salaisuudet ja asetukset (`root:chatapp`, `640`) |
| `/run/chatapp/gunicorn.sock` | Gunicornin Unix-soketti |
| `/etc/nginx/snippets/chatapp.conf` | Nginx-snippet |
| `/etc/systemd/system/chatapp.service` | systemd-palvelu |

Sovellus ajetaan järjestelmäkäyttäjänä `chatapp` (ei kirjautumista). Tietokantakäyttäjällä `chatapp` on oikeudet vain tietokantaan `chatapp`.