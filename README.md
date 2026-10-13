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

**Variables** (opsional, tab Variables). Default-nya sudah untuk `mardhotullah.com`;
isi hanya kalau ingin mengganti. Nilai ini di-bake ke image web saat build, jadi ganti nilai = build ulang.

| Nama | Default |
|---|---|
| `NEXT_PUBLIC_APP_URL` | `https://mardhotullah.com` |
| `NEXT_PUBLIC_LOGIN_URL` | `https://cms.mardhotullah.com/login` |
| `NEXT_PUBLIC_PPDB_URL` | (kosong) |
| `NEXT_PUBLIC_WHATSAPP_NUMBER` | (kosong) |

CMS dan web memanggil backend lewat proxy di domainnya sendiri (`/api/backend` dan `/api/publik`),
jadi tidak perlu konfigurasi URL API.

### Trigger dari repo aplikasi (opsional)

```bash
curl -X POST \
  -H "Authorization: Bearer <PAT dengan akses Actions/Contents write ke repo infra>" \
  https://api.github.com/repos/<owner>/ponpes-mardhotullah-infrastructure/dispatches \
  -d '{"event_type":"build","client_payload":{"service":"backend","ref":"main"}}'
```

## Deploy dengan Docker Compose + Traefik

| URL | Service |
|---|---|
| https://mardhotullah.com | web (`www.` di-redirect ke sini) |
| https://cms.mardhotullah.com | cms |
| https://api.mardhotullah.com | backend |

Traefik menangani HTTPS otomatis (Let's Encrypt, HTTP challenge) dan redirect HTTP → HTTPS.

1. Arahkan DNS **A record** `mardhotullah.com`, `www`, `cms`, `api` ke IP server.
2. Buka port **80** dan **443** di firewall server.
3. Jalankan:

```bash
cp .env.example .env        # isi password, secret, ACME_EMAIL
echo <PAT read:packages> | docker login ghcr.io -u <owner> --password-stdin   # jika package private
docker compose pull
docker compose up -d
```

Backend menjalankan migrasi & seeder otomatis saat start. Bucket MinIO dibuat otomatis.
Console MinIO hanya listen di `127.0.0.1:9001` server — akses lewat `ssh -L 9001:localhost:9001 <server>`.
