# vitakriz-app-template

Šablona pro nový web nebo appku, která se po `git push` sama nasadí na VPS
`vps-01.vitakriz.cz` (Cloudflare Tunnel → caddy-docker-proxy → Docker) bez
ručního zásahu v Caddy nebo Portaineru.

Hodí se pro:

- **statický web** — bez databáze, bez `.env`, jen nginx s HTML/CSS/JS,
- **appku bez databáze** — server, volitelně s API klíči v `.env`,
- **appku s PostgreSQL** — jen když ji appka opravdu potřebuje.

A pro libovolnou doménu: `foo.vitakriz.cz` funguje hned, jiné domény (apex
jako `ekotex.eu` i subdomény jako `www.ekotex.eu`) po jednorázovém nastavení
DNS a tunelu.

Kompletní vysvětlení architektury a přesný postup je v [`CLAUDE.md`](./CLAUDE.md)
— dej tenhle soubor svému Claude Code na lokálním stroji a řekni mu, ať podle
něj založí nový projekt.

## Obsah

| Soubor | K čemu |
|---|---|
| `docker-compose.static.yml` | statický web (nginx, port 80, bez `.env`) |
| `docker-compose.yml` | appka bez databáze (`.env` volitelný) |
| `docker-compose.with-db.yml` | appka s PostgreSQL |
| `Dockerfile.static.example` | nginx — čisté HTML, nebo build přes Bun (Vite/Astro…) |
| `Dockerfile.app.example` | serverová appka (Bun) |
| `.github/workflows/deploy.yml` | build → GHCR → deploy na VPS (stejný pro všechny typy) |
| `.env.example` | vzor `.env` — **nikdy** se necommituje, vytváří se ručně na VPS |

Z compose šablon si vyber jednu a ulož ji jako `docker-compose.yml`
(ostatní smaž). Totéž u Dockerfilu.

## Minimální kroky

1. Zkopíruj obsah repa (nebo "Use this template" na GitHubu) do nového repa.
2. Vyber compose šablonu a doplň `ghcr.io/OWNER/REPO`, `HOSTNAME` a `PORT`.
3. V `.github/workflows/deploy.yml` nastav `APP_NAME:` (jen `[a-z0-9-]`).
4. Přidej `Dockerfile` (server musí poslouchat na `0.0.0.0:<PORT>`).
5. GitHub repo → Settings → Secrets and variables → Actions: `VPS_HOST`,
   `VPS_DEPLOY_KEY`.
6. Hostname mimo `*.vitakriz.cz`? Nastav DNS CNAME na tunel a ingress
   pravidlo na VPS — viz "Jiná doména" v `CLAUDE.md`.
7. Potřebuje appka `.env`? Po prvním pushi ho vytvoř na VPS v
   `/opt/apps/<APP_NAME>/.env` a workflow spusť znovu. Statický web tenhle
   krok přeskakuje.
8. `git push` — web je do minuty na `https://<HOSTNAME>`.

## Privátní repo / GHCR image

Appka nemusí být veřejná — uživatel `deploy` na VPS už má nastavené
přihlášení k `ghcr.io` (scope `read:packages`), takže `docker compose pull`
funguje i pro privátní balíčky. Šablony se nijak nemění, viz `CLAUDE.md`.
