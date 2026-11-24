<div align="center">

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="app/spa/public/favicon.png">
  <img src="app/spa/public/icons/favicon-source.png" alt="Logo Berthu" width="120">
</picture>

# Berth

**Vlastní PaaS pro vaše servery.**
Nasazení bez výpadku, databáze se zálohami, monitoring a registr image
na serverech, které máte pod kontrolou.

[![Vydání](https://img.shields.io/github/v/release/linuxak-dev/berth?label=vyd%C3%A1n%C3%AD&color=2563eb)](https://github.com/linuxak-dev/berth/releases/latest)
![Platforma](https://img.shields.io/badge/Rocky%20%7C%20RHEL-10-10b981)
![Podman](https://img.shields.io/badge/Podman-Quadlety-892ca0)
![Caddy](https://img.shields.io/badge/HTTPS-Caddy-1f88c0)

[Rychlý start](#-rychlý-start) ·
[Co Berth umí](#-co-berth-umí) ·
[CLI](#-cli) ·
[Instalace](#-instalace-centrály) ·
[Vydání](https://github.com/linuxak-dev/berth/releases)

</div>

---

Berth dává vlastním serverům pohodlí platforem typu Heroku nebo Render,
ale data, servery i náklady zůstávají u vás. Projekt popíše své služby
v jednom souboru `deploy/deploy.yaml`, CI nebo CLI ho pošle centrále
a agent na serveru nasadí služby jako Quadlety Podmanu za Caddy.

> [!TIP]
> Aplikace na Berthu nezávisí. Služby běží jako systémové jednotky
> Podmanu, takže když centrála zrovna neběží, aplikace, zálohy i HTTPS
> jedou dál.

## 🚀 Rychlý start

**1. Centrála** na čistém serveru s Rocky nebo RHEL 10 (jako root):

```bash
curl -fsSL https://get.linuxak.com/berth | sh
```

**2. Manifest** v repozitáři aplikace, `deploy/deploy.yaml`:

```yaml
version: 2
project: Shop
slug: shop
server: local

services:
  web:
    image: registry.example.com/shop/web
    build:
      context: .
      dockerfile: Dockerfile
    port: 8080
    health: /health
    environment:
      DATABASE_URL: "postgres://app:${POSTGRES_PASSWORD}@{db:postgres}:5432/shop?sslmode=disable"
    routes:
      - hosts:
          - shop.example.com

databases:
  postgres:
    type: postgres
    image: docker.io/library/postgres
    tag: "18"
    database: shop
    user: app
    password: POSTGRES_PASSWORD

environments:
  production:
    branch: main
```

**3. Nasazení** z počítače nebo z CI:

```bash
berth login https://berth.example.com
berth deploy production --tag 1a2b3c4d
```

## ✨ Co Berth umí

| | |
|---|---|
| 🔄 **Nasazení bez výpadku** | blue/green s kontrolou zdraví, automatický rollback, každý release jde vrátit jedním kliknutím |
| 🔐 **Tajné údaje v gitu** | šifrované `sops` a `age`, klíč na prostředí, šifruje centrála |
| 🐘 **Databáze** | Postgres v manifestu, zálohy na S3, obnova z dumpu, Adminer v prohlížeči |
| 🖥️ **Provoz z prohlížeče** | log, terminál do kontejneru, správce souborů, ovládání služeb |
| 📈 **Monitoring a alerty** | grafy serverů a služeb, logy, alerty e-mailem a do Telegramu |
| 🌐 **Víc serverů** | připojení přes SSH jedním formulářem, WireGuard a mTLS |
| 📦 **Registr image** | vestavěný zot s uživateli, retencí a skenem zranitelností |
| 🛡️ **Bezpečnost** | rootless Podman bez capabilities, audit, role a přístupy po prostředích |
| ♻️ **Centrála se stará sama** | instalace jedním příkazem, aktualizace bez výpadku, noční šifrovaná záloha |

### 🔄 Nasazení bez výpadku a bez strachu

- **Celá aplikace v jednom souboru.** `deploy/deploy.yaml` popisuje
  služby, databáze, domény, routy, zálohy a rozdíly mezi prostředími.
  Berth ho před nasazením přísně zkontroluje a chybu ohlásí dřív, než se
  cokoli dotkne serveru.
- **Blue/green.** Nová verze naběhne vedle staré, projde kontrolou zdraví
  a teprve pak na ni Caddy přepne provoz. Služba, která dvě kopie vedle
  sebe nesnese, má `strategy: recreate`.
- **Automatický rollback.** Když nová verze po přepnutí spadne, Berth sám
  vrátí předchozí. Ručně jedním kliknutím nebo `berth rollback`.
- **Z CI jedním krokem.** Sdílený workflow pro Gitea Actions postaví image,
  pošle je do registru a nasadí, průběh je živě vidět v terminálu i v UI.
- **HTTPS a DNS samo.** Let's Encrypt nebo Cloudflare DNS (i pro servery
  na soukromé adrese), záznamy A u Cloudflare se založí při nasazení.
  Routy umí přesměrování, kanonickou doménu, cesty, omezení na adresy,
  Basic Auth i `noindex`.
- **Nejen web.** Veřejné TCP porty (třeba SSH Gitey), zařízení hostitele
  (`/dev/fuse` pro buildah) a limity CPU a paměti. Na firewallu se
  otevírají jen jednotlivé porty, nikdy rozsah do zásoby.

### 🔐 Bezpečné tajné údaje

- Hesla a klíče leží v gitu šifrované ve formátu `sops` s klíči `age`,
  každé prostředí má vlastní klíč a záchranného příjemce.
- `berth secret set` šifruje přes centrálu, nikdo nepotřebuje `sops` ani
  klíč u sebe. Změna jedné hodnoty mění v gitu jen jeden řádek.
- Tajné hodnoty se nikdy nezapisují do logu, událostí ani do popisu
  releasu. Výměna klíče je jeden příkaz `berth secret rekey`.

### 🐘 Databáze bez starostí

- **Postgres jako součást aplikace**, stačí ho popsat v manifestu.
- **Zálohy na S3** podle cronu s retencí, volitelně i před nasazením,
  které databázi mění. Běží i bez centrály.
- **Obnova** ze zálohy nebo nahraného dumpu v jedné transakci, takže
  vadný dump databázi nepoškodí. Velké dumpy se nahrávají po částech
  s navázáním po výpadku spojení.
- **Vyprázdnění dat** jedním kliknutím: služba se spustí načisto,
  Postgres s novým `initdb`.
- **Adminer v prohlížeči** s rovnou přihlášenou databází a
  `berth db tunnel` pro připojení z vlastního počítače.

### 🖥️ Provoz aplikací z prohlížeče

- Stav služeb, log, spotřeba, porty, volumes a soubor Quadletu na jednom
  místě.
- **Terminál do kontejneru** v prohlížeči, zastavení, spuštění a restart
  služby jedním kliknutím.
- **Správce souborů** ve volumes: procházení, nahrání i přetažením
  (archiv se rozbalí na serveru), přesun, přejmenování, stažení, mazání.
- **Odebrání prostředí** se vším, co na serveru má, na přání i s daty.
  Úklid dat po odebraných službách a nepoužívaných image.
- Služba s `autoUpdate` se sama aktualizuje z registru a UI to označí.
- **Hlídání nových verzí image**: jednou denně Berth zjistí, jestli
  k image služeb, databází a základům buildů (`FROM`, `COPY --from`
  v Containerfile) vyšla novější verze, a pošle souhrnný e-mail. Velké
  verze označí zvlášť, povýšení zůstává ruční.

### 📈 Monitoring a alerty

- **Vestavěný monitoring** bez Grafany a Promethea: grafy serverů (CPU,
  paměť, disky, síť) a služeb (CPU, paměť, restarty, OOM, disk, síť).
- **Logy všech služeb** na jednom místě s filtry, histogramem a živým
  sledováním.
- **Alerty** s výchozími i vlastními pravidly, upozornění e-mailem nebo do
  Telegramu, ztlumení na dobu údržby a hlídač samotného monitoringu.
- **Nástěnka „Vyžaduje pozornost“**: nedostupný server, neběžící služba,
  selhané nasazení nebo záloha, docházející disk, kritické CVE, nová
  verze Berthu.

### 🌐 Víc serverů, jedna centrála

- Vzdálený server se připojí formulářem v UI. Berth na něj přes SSH
  nainstaluje agenta, Podman i Caddy a spojí ho s centrálou přes
  WireGuard a mTLS.
- Přehled serverů s využitím zdrojů, aktualizace agentů a Caddy, rotace
  SSH klíčů jedním tlačítkem.
- Přístupy k registrům, tokeny DNS a cíle S3 se samy rozešlou na všechny
  servery, i na ty, které byly zrovna nedostupné.

### 📦 Vlastní registr image

- Vestavěný registr **zot** s procházením repozitářů a tagů, časem
  nahrání, počtem stažení a informací, kde se image používá.
- **Uživatelé a práva** po repozitářích a **retence** s náhledem, co by
  pravidlo smazalo.
- **Sken zranitelností** (Trivy) s přehledem CVE u každého tagu.

### 👥 Týmy a přístupy

- Organizace, uživatelé a role s přehledem, co která role smí.
- Přístup po prostředích ve třech úrovních: čtení, nasazení, tajné údaje.
- Osobní tokeny pro CLI s přihlášením přes prohlížeč a nasazovací tokeny
  pro CI omezené na projekt nebo prostředí.
- Audit citlivých akcí a upozornění e-mailem s výběrem příjemců.

### ♻️ Centrála, která se stará sama o sebe

- **Instalace jedním příkazem**, s doménou i bez ní.
- **Aktualizace bez výpadku** z UI nebo `berth server update`, se zálohou
  předem a návratem přes `berth server repair`.
- **Noční šifrovaná záloha** na S3 a obnova na novém serveru příkazem
  `berth server restore`. Servery se k obnovené centrále připojí samy.
- Doména, certifikát, pošta, registr, monitoring i firewall se nastaví
  v UI nebo jedním příkazem `berth server …`.
- Příručka přímo v aplikaci, otazník u každé stránky vede na její oddíl.

## ⌨️ CLI

<details open>
<summary><b>Každodenní práce</b></summary>

```bash
berth login https://berth.example.com   # přihlášení přes prohlížeč
berth init                              # připraví repo (.sops.yaml, .gitignore)
berth validate                          # kontrola manifestu bez serveru
berth deploy staging --tag 1a2b3c4d     # nasazení s živým průběhem
berth status staging                    # stav služeb
berth logs staging web                  # log služby
berth rollback staging                  # návrat k předchozí verzi
berth service restart staging web       # restart služby
```

</details>

<details>
<summary><b>Tajné údaje, databáze a soubory</b></summary>

```bash
berth secret set staging env.API_KEY             # tajný údaj bez sops
berth secret rekey staging                       # výměna klíče prostředí
berth db tunnel staging main                     # databáze na localhostu
berth db import staging main dump.sql --confirm main   # obnova z dumpu
berth files put staging ./uploads web:data/ -r   # soubory do volume
berth files put staging data.tar web:data --extract --preserve   # archiv s právy a vlastníky
berth files get staging web:data/export backup.tar               # stažení z volume
```

</details>

<details>
<summary><b>Monitoring a správa centrály</b></summary>

```bash
berth monitoring alerts                          # aktivní alerty
berth monitoring silence --alert DiskSkoroPlny --for 2h --node web1
berth server update --version 0.7.4              # aktualizace centrály
berth server firewall --open 20202/tcp           # jeden veřejný port
berth server domain --domain berth.example.com --cert letsencrypt
berth server backup                              # záloha centrály teď
```

</details>

## 📥 Instalace centrály

Na čistém serveru s **Rocky Linux** nebo **RHEL 10** jako root:

```bash
curl -fsSL https://get.linuxak.com/berth | sh
```

Instalace se zeptá na adresu serveru (jen když jich najde víc) a na účet
administrátora. UI pak běží na `https://<adresa>/berth` s interním
certifikátem. Doménu a důvěryhodný certifikát (Let's Encrypt, nebo
Cloudflare DNS pro server na soukromé adrese) nastavíte v UI na stránce
Centrála nebo příkazem `berth server domain`.

<details>
<summary><b>Bezobslužná instalace</b></summary>

Volby jdou za `sh -s --`:

```bash
curl -fsSL https://get.linuxak.com/berth | sh -s -- \
  --domain berth.example.com \
  --cert cloudflare --cloudflare-token-file /root/cf.token \
  --admin-email admin@example.com --admin-password-file /root/admin.pw
```

Všechny volby vypíše `berth server install --help`.

</details>

Binárky CLI pro **Linux**, **macOS** a **Windows** jsou u každého vydání
na [GitHub Releases](https://github.com/linuxak-dev/berth/releases).

## 🧱 Jak to funguje

```text
  git push ──► CI ──► berth deploy ──► centrála (API + UI)
                                          │  mTLS přes WireGuard
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
               agent uzlu            agent uzlu            agent uzlu
           Quadlety · Caddy      Quadlety · Caddy      Quadlety · Caddy
           Postgres · zálohy     Postgres · zálohy     Postgres · zálohy
```

- **Centrála** drží projekty, prostředí, tajné údaje a historii nasazení,
  běží v ní webové rozhraní.
- **Agent** na každém serveru nasazuje služby, sbírá metriky a logy
  a spouští zálohy. Rozhoduje sám, centrála mu jen posílá popis releasu.
- **Caddy** obsluhuje HTTPS a routy všech prostředí na serveru.

---

<details>
<summary><b>Pro vývojáře Berthu</b></summary>

| Část | Co to je |
|---|---|
| `app/api` | API centrály (jádro Plinth), CLI `berth`, agent `berth-agent` |
| `app/spa` | webové rozhraní centrály |
| `caddy` | build Caddy s modulem Cloudflare (vlastní vydání `caddy-<verze>`) |
| `examples/hello` | testovací aplikace |
| `dev/stack` | lokální Gitea, runner a zot pro vývoj |

```bash
make stack-up          # lokální Gitea a zot
make test              # testy a kontroly
make build-cli         # bin/berth
make build-linux       # CLI a agent pro linux/arm64
```

Návrh a postup implementace: `.thoughts/shared/plans/2026-11-08-berth.md`.

</details>
