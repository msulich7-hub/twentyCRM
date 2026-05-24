# twentyCRM

Lokalna kopia [Twenty CRM](https://github.com/twentyhq/twenty) — open-source CRM (NestJS + React + PostgreSQL + Redis).

## Pobranie / aktualizacja

```bash
git submodule update --init --recursive
```

## Uruchomienie (Docker)

```bash
cd twenty/packages/twenty-docker
docker compose up -d
```

## Uruchomienie (development)

Wymagania: Node `^24.5.0`, Yarn `>=4.0.2`, PostgreSQL, Redis.

```bash
cd twenty
yarn install
yarn start
```

Dokumentacja: https://docs.twenty.com
