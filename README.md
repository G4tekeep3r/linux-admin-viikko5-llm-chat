# LLM-chat cPouta (CI/CD-harjoitus, viikko 5)
 
Web-chat, jossa käyttäjä keskustelee palvelimella paikallisesti pyörivän kielimallin (Ollama) kanssa. Sovellus (Flask + Gunicorn) on Nginxin takana osoitteessa `/viikko5/`, keskusteluhistoria tallennetaan MariaDB:hen, ja jokainen push `main`-haaraan testataan ja julkaistaan GitHub Actionsilla.
 
## Repositorion rakenne
 
```
.
├── app.py                      # Flask: /, /health, /docs, /api/chat, /api/history/<id>
├── db.py                       # MariaDB-kerros
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
 
1. Kopioi ja aja alustusskripti palvelimella:
   ```bash
   scp scripts/server-setup.sh ubuntu@<floating-ip>:~
   ssh ubuntu@<floating-ip>
   bash server-setup.sh qwen2.5:0.5b
   ```
   Skripti tarkistaa resurssit ja portit, asentaa puuttuvat paketit, luo käyttäjän `chatapp`, tietokannan ja salaisuustiedoston `/etc/chatapp/chatapp.env`, asentaa Ollaman ja lataa mallin. Tietokannan salasana arvotaan palvelimella.
 
2. Lisää Nginx-sivuston `server { }`-lohkoon rivi `include snippets/chatapp.conf;` ja ota käyttöön:
   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```
 
## Deploy-avain ja GitHub Secrets
 
GitHub Actions käyttää omaa SSH-avainparia (ei henkilökohtaista avainta):
 
```powershell
ssh-keygen -t ed25519 -f $HOME\.ssh\cpouta_deploy -C "github-actions-deploy"
```
 
Julkinen avain lisätään palvelimen `~/.ssh/authorized_keys`-tiedostoon. Settings → Secrets and variables → Actions:
 
| Secret | Sisältö |
|---|---|
| `CPOUTA_HOST` | palvelimen floating IP |
| `CPOUTA_USER` | `ubuntu` |
| `CPOUTA_SSH_KEY` | deploy-avaimen yksityinen avain |
| `CPOUTA_KNOWN_HOSTS` | palvelimen host key (`ssh-keygen -F <ip>`) |
 
Avaimia, `.env`-tiedostoja tai salasanoja ei viedä repositorioon.