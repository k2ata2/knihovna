# VPS infrastruktura (vps-01.vitakriz.cz) — jak vytvořit novou appku / web

Tento soubor popisuje infrastrukturu VPS (`vps-01.vitakriz.cz`) a přesný postup,
jak založit nový projekt tak, aby se sám nasazoval po `git push` bez ručního
zásahu v Caddy nebo Portaineru.

Na VPS běží cokoliv, co jde zabalit do Docker image:

- **statický web** (HTML/CSS/JS, výstup Vite/Astro/Hugo buildu, …) — žádná
  databáze, žádný `.env`, typicky nginx na portu 80,
- **appka bez databáze** (Node/Bun/Python/… server, případně s API klíči v `.env`),
- **appka s PostgreSQL databází**.

Databáze **není** povinná — přidává se jen tehdy, když ji appka opravdu
potřebuje. A hostname nemusí být pod `vitakriz.cz` — viz sekce o doménách.

## Architektura

```
Internet
  → Cloudflare (DNS proxy, Universal SSL)
  → Cloudflare Tunnel "vitakriz-apps" (locally-managed, jen config.yml na VPS,
    NIC v Cloudflare dashboardu)
  → caddy-docker-proxy (Docker kontejner na VPS, localhost:8088, sleduje
    Docker labels)
  → Docker síť "proxy" (external, 10.0.14.0/24)
  → kontejner appky (Host-header routing podle Docker labelu `caddy:`)
```

Routing jede čistě podle Host hlavičky — stejný tunel a stejný Caddy
obsluhují libovolný počet domén i subdomén.

Deployment: `git push` do `main`/`master` → GitHub Actions zabuildí image,
pushne na `ghcr.io`, přes omezený SSH klíč pošle `docker-compose.yml` na VPS
a spustí `docker compose pull && up -d --remove-orphans`. Žádný Portainer
stack, žádný GitHub token na serveru, žádné ruční přidělování portu.

## Domény — jaký hostname může appka mít

| Hostname | Příklad | Co je potřeba mimo repo |
|---|---|---|
| jednoúrovňová subdoména `vitakriz.cz` | `foo.vitakriz.cz` | **nic** (wildcard) |
| apex cizí domény (top-level) | `ekotex.eu` | DNS záznam + ingress pravidlo na VPS |
| subdoména cizí domény | `shop.ekotex.eu`, `www.ekotex.eu` | DNS záznam + ingress pravidlo na VPS |
| apex `vitakriz.cz` | `vitakriz.cz` | jako cizí doména (wildcard apex nepokrývá) |
| víceúrovňová subdoména | `foo.apps.vitakriz.cz` | **NEPOUŽÍVAT** (placený Total TLS) |

### `<neco>.vitakriz.cz` — bez jakékoliv práce

Cloudflare DNS má wildcard `*.vitakriz.cz` → tunel `vitakriz-apps`. Přesný
DNS záznam vždy vyhrává nad wildcardem, takže wildcard **nemůže** přepsat
žádný existující hostname — a naopak **jakýkoliv nový, dosud nepoužitý
`<cokoliv>.vitakriz.cz` je automaticky funkční**, stačí správný `caddy:`
label.

Wildcard je jednoúrovňový (`*.vitakriz.cz`), NE `*.apps.vitakriz.cz`.
Víceúrovňové subdomény by potřebovaly placený Cloudflare Total TLS — záměrně
se tomu vyhýbáme.

### Jiná doména — apex (top-level) i subdomény

Ano, funguje to i na holou doménu (`ekotex.eu`) i na její subdomény. Postup:

1. Doména musí mít nameservery na Cloudflare (vlastní zóna v účtu).
2. V DNS zóně té domény vytvořit **proxied** (oranžový mráček) CNAME na
   tunel `vitakriz-apps` (`<TUNNEL_ID>.cfargotunnel.com`):
   - apex: `ekotex.eu` → CNAME na apexu funguje díky Cloudflare CNAME
     flatteningu,
   - subdoména: `www.ekotex.eu`, `shop.ekotex.eu`, … (případně wildcard
     `*.ekotex.eu`, pokud jich bude víc).
3. Do `/etc/cloudflared-apps/config.yml` na VPS přidat ingress pravidlo
   (před závěrečné catch-all pravidlo):
   ```yaml
   - hostname: ekotex.eu
     service: http://localhost:8088
   - hostname: "*.ekotex.eu"       # volitelně, pokryje všechny subdomény
     service: http://localhost:8088
   ```
   a `systemctl restart cloudflared-apps`. Tohle umí jen admin (nebo Claude
   se SSH přístupem) na VPS, **ne CI**.
4. Appka pak má normálně label `caddy: http://ekotex.eu` — beze změny
   oproti vitakriz.cz appkám. Více hostnamů pro jednu appku se oddělí
   čárkou: `caddy: http://ekotex.eu, http://www.ekotex.eu`.
5. TLS je zdarma přes Universal SSL té domény (pokrývá apex + jednu úroveň
   subdomén, Total TLS není potřeba).

Stav: pro `ekotex.eu` je krok 3 na VPS už hotový (ingress pravidlo
existuje), chybí jen DNS záznam z kroku 2.

## Co Claude (na tomto lokálním stroji) má udělat, když uživatel řekne
"založ novou appku / web pro vitakriz infrastrukturu" / podobně

1. Zeptej se (pokud to není z kontextu jasné):
   - název appky — bude i název GitHub repa a adresáře na VPS, jen
     `[a-z0-9-]`,
   - hostname(y) — `<neco>.vitakriz.cz`, nebo jiná doména (apex/subdoména),
   - **typ**: statický web / appka bez DB / appka s PostgreSQL. Neptej se
     na DB u zjevně statického webu a DB nikdy nepřidávej "pro jistotu".
   - interní port kontejneru (u statického webu s nginx je to `80`).
2. Vytvoř `Dockerfile` podle stacku (vzory: `Dockerfile.static.example`,
   `Dockerfile.app.example` v tomto repu). Server v kontejneru musí
   poslouchat na `0.0.0.0:<PORT>`, ne jen na `localhost`.
3. Vytvoř `docker-compose.yml` podle příslušné šablony níže.
4. `.env.example` (bez skutečných hodnot) — jen pokud appka nějaké
   proměnné potřebuje. Statický web ho nepotřebuje.
5. Vytvoř `.github/workflows/deploy.yml` podle šablony níže, uprav
   `APP_NAME:`.
6. Přidej `.gitignore` s `.env` uvnitř — **`.env` se nikdy necommituje.**
7. Řekni uživateli přesně tyto ruční kroky (Claude je sám udělat nemůže):
   - V GitHub repu → Settings → Secrets and variables → Actions přidat:
     - `VPS_HOST` = `vps-01.vitakriz.cz`
     - `VPS_DEPLOY_KEY` = obsah privátního klíče (uživatel ho má uložený
       lokálně z VPS migrace, typicky `~/.ssh/vitakriz-deploy` nebo podobně —
       zeptej se, pokud ho nemá po ruce)
   - **Jen pokud hostname není `<neco>.vitakriz.cz`:** DNS záznam +
     ingress pravidlo podle sekce "Jiná doména" výše.
   - **Jen pokud appka potřebuje `.env`** (DB heslo, API klíče): po prvním
     pushi (který vytvoří `/opt/apps/<APP_NAME>/` na VPS) se přihlásit přes
     SSH jako `vita` a vytvořit `/opt/apps/<APP_NAME>/.env` ručně
     (`sudo -u deploy nano /opt/apps/<APP_NAME>/.env` nebo `sudo cp` +
     `chown deploy:deploy`), pak znovu spustit workflow (Actions → Re-run).
     Bez `.env` appka s DB nenaběhne.
   - `git push` do `main`/`master` → sleduj Actions tab, appka by měla být
     do minuty dostupná na `https://<hostname>`.

## Šablona: docker-compose.yml (statický web)

```yaml
services:
  app:
    image: ghcr.io/OWNER/REPO:latest
    restart: unless-stopped
    networks:
      - proxy
    labels:
      caddy: http://HOSTNAME
      caddy.reverse_proxy: "{{upstreams 80}}"

networks:
  proxy:
    external: true
```

## Šablona: docker-compose.yml (appka BEZ databáze)

```yaml
services:
  app:
    image: ghcr.io/OWNER/REPO:latest
    restart: unless-stopped
    env_file:
      - path: .env
        required: false   # appka naběhne i bez .env na VPS
    networks:
      - proxy
    labels:
      caddy: http://HOSTNAME
      caddy.reverse_proxy: "{{upstreams PORT}}"

networks:
  proxy:
    external: true
```

## Šablona: docker-compose.yml (appka S PostgreSQL databází)

```yaml
services:
  app:
    image: ghcr.io/OWNER/REPO:latest
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - default
      - internal-db
      - proxy
    labels:
      caddy: http://HOSTNAME
      caddy.reverse_proxy: "{{upstreams PORT}}"

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 20
    networks:
      - internal-db

volumes:
  postgres-data:

networks:
  internal-db:
    internal: true
  proxy:
    external: true
```

Databáze je **vždy** jen v `internal-db` (interní síť, žádný publikovaný
port) a `proxy` síť má **jen** služba `app`. Tohle nikdy neměň.

`HOSTNAME` = např. `foo.vitakriz.cz`, `ekotex.eu`, nebo víc hostnamů
oddělených čárkou (`http://ekotex.eu, http://www.ekotex.eu`). Vždy s
prefixem `http://` — TLS ukončuje Cloudflare, Caddy za tunelem jede jen
HTTP a nesmí si zkoušet vystavovat vlastní certifikát.

## Šablona: .github/workflows/deploy.yml

```yaml
name: Deploy

on:
  push:
    branches: [main]   # nebo master
  workflow_dispatch:

env:
  APP_NAME: CHANGE_ME
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE }}:latest
            ${{ env.IMAGE }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Sync docker-compose.yml to VPS
        run: |
          mkdir -p ~/.ssh
          printf '%s\n' "${{ secrets.VPS_DEPLOY_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          rsync -av --mkpath \
            -e "ssh -i ~/.ssh/deploy_key -o StrictHostKeyChecking=accept-new" \
            docker-compose.yml \
            deploy@${{ secrets.VPS_HOST }}:${{ env.APP_NAME }}/docker-compose.yml

      - name: Trigger deploy on VPS
        run: |
          ssh -i ~/.ssh/deploy_key -o StrictHostKeyChecking=accept-new \
            deploy@${{ secrets.VPS_HOST }} "deploy ${{ env.APP_NAME }}"
```

Workflow je stejný pro všechny tři typy (statický web, appka, appka s DB).

## Jak deployment mechanismus na VPS funguje (pro pochopení, needituj)

- Na VPS existuje omezený systémový uživatel `deploy` (member skupiny
  `docker`, shell `/bin/bash`, ale žádné heslo, žádný interaktivní login).
- Jeho `~/.ssh/authorized_keys` má **forced command** — ať GitHub Actions
  pošle přes SSH cokoliv, na serveru se vždy spustí jen
  `/opt/deploy/bin/gh-deploy-dispatch.sh`, který povoluje výhradně:
  - `rsync` zápis (přes `rrsync -wo`) omezený na `/opt/apps/`
  - příkaz `deploy <app-name>` → `cd /opt/apps/<app-name> && docker compose
    pull && docker compose up -d --remove-orphans`, jen pokud tam už
    existuje `docker-compose.yml` a jméno appky projde regexem
    `^[a-z0-9][a-z0-9-]{0,62}$`
- Cesty v rsync příkazu musí být **relativní** k `/opt/apps` (rrsync se
  chová jako chroot) — proto v workflow šabloně výše cesty typu
  `${{ env.APP_NAME }}/docker-compose.yml`, ne `/opt/apps/...`.
- `.env` soubory se nikdy nepřenáší přes CI — žijí jen na VPS, vytváří se
  ručně jednou při prvním nasazení appky (a jen pokud je appka potřebuje).
- CI umí měnit jen obsah `/opt/apps/`. Cloudflare DNS a
  `/etc/cloudflared-apps/config.yml` (ingress pro jiné domény) přes CI
  změnit nejde.

## Privátní GHCR image (repo nemusí být veřejné)

Appka **nemusí** být ve veřejném GitHub repu ani mít veřejný GHCR image —
šablona `docker-compose.yml`/`deploy.yml` se v tom případě nijak nemění.

- Uživatel `deploy` na VPS má už jednorázově uložené přihlášení k
  `ghcr.io` (token se scope `read:packages`), takže `docker compose pull`
  zvládne stáhnout privátní image stejně snadno jako veřejný — nic
  navíc není potřeba nastavovat pro každé nové repo zvlášť.
- GHCR balíček (image) automaticky dědí viditelnost ze zdrojového repa —
  privátní repo → privátní balíček, není potřeba nic extra zaškrtávat.
- Build/push krok (`docker/build-push-action` s `secrets.GITHUB_TOKEN`)
  funguje identicky pro veřejné i privátní repo.

Jediné, co by vyžadovalo zásah na VPS: pokud by appka byla v **jiném**
GitHub účtu/organizaci než `sprtokiller` (jiný vlastník balíčku) — pak by
bylo potřeba nový `docker login ghcr.io` s tokenem platným pro ten účet.

## Co NIKDY nedělat

- Necommituj `.env`, tokeny, privátní klíče do gitu.
- Nezakládej appku pod víceúrovňovou subdoménou (`*.apps.vitakriz.cz` apod.)
  — vyžadovalo by placený Cloudflare Total TLS, záměrně se tomu vyhýbáme.
- Nepřidávej appce publikovaný host port (`ports:` s `host:container`) —
  routing jde výhradně přes `proxy` síť + Caddy labels.
- Databázový kontejner nikdy nepřipojuj do sítě `proxy`.
- Nepřidávej databázi appce, která ji nepotřebuje (statický web apod.).
- Nepoužívej v `caddy:` labelu `https://` ani holý hostname bez `http://`
  — Caddy by zkoušel získat certifikát, což za tunelem nejde.
- Nepředpokládej, že máš přístup ke starému Portainer/Caddy/Cloudflare
  dashboard postupu — ten byl nahrazen tímto systémem a nepoužívá se.
