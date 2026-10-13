# ponpes-mardhotullah-infrastructure

Build image Docker dan konfigurasi deployment untuk:

| Service | Repo | Image |
|---|---|---|
| backend | `ponpes-mardhotullah-backend` | `ghcr.io/<owner>/ponpes-backend` |
| cms | `ponpes-mardhotullah-cms` | `ghcr.io/<owner>/ponpes-cms` |
| web | `ponpes-mardhotullah-web` | `ghcr.io/<owner>/ponpes-web` |

## GitHub Actions: Build & Push Images

Workflow [`.github/workflows/build-images.yml`](.github/workflows/build-images.yml) checkout repo aplikasi,
build pakai `Dockerfile` di repo tersebut, lalu push ke GHCR dengan tag `<sha pendek>` dan `latest`
(tag `latest` hanya kalau build dari default branch).

Jalankan dari tab **Actions → Build & Push Images → Run workflow**, pilih service (`all`/`backend`/`cms`/`web`)
dan opsional `ref` (branch/tag/SHA).

### Setup sekali

**Secrets** (Settings → Secrets and variables → Actions → Secrets)

| Nama | Isi |
|---|---|
| `REPO_TOKEN` | Fine-grained PAT dengan akses **Contents: Read** ke 3 repo aplikasi. Wajib kalau repo aplikasi private. |

**Variables** (tab Variables). Nilai `NEXT_PUBLIC_*` di-bake ke image saat build, jadi ganti nilai = build ulang.

| Nama | Contoh | Dipakai |
|---|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | `https://api.example.com/api` | cms, web (wajib) |
| `API_BASE_URL` | `http://backend:8080/api` | cms, web (rewrite `/uploads`, `/api/publik`) |
| `NEXT_PUBLIC_APP_URL` | `https://example.com` | web |
| `NEXT_PUBLIC_LOGIN_URL` | `https://cms.example.com/login` | web |
| `NEXT_PUBLIC_PPDB_URL` | | web |
| `NEXT_PUBLIC_WHATSAPP_NUMBER` | `628xxxx` | web |

### Trigger dari repo aplikasi (opsional)

```bash
curl -X POST \
  -H "Authorization: Bearer <PAT dengan akses Actions/Contents write ke repo infra>" \
  https://api.github.com/repos/<owner>/ponpes-mardhotullah-infrastructure/dispatches \
  -d '{"event_type":"build","client_payload":{"service":"backend","ref":"main"}}'
```

## Deploy dengan Docker Compose

```bash
cp .env.example .env        # isi password & secret
echo <PAT read:packages> | docker login ghcr.io -u <owner> --password-stdin   # jika package private
docker compose pull
docker compose up -d
```

Backend menjalankan migrasi & seeder otomatis saat start. Bucket MinIO dibuat otomatis.
Pasang reverse proxy (Nginx/Caddy) + TLS di depan port backend, cms, dan web.
