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
(tag `latest` hanya kalau yang dibuild adalah default branch repo tersebut).

Jalankan dari tab **Actions → Build & Push Images → Run workflow**, pilih service (`all`/`backend`/`cms`/`web`)
dan opsional `ref`.

| Repo | Default branch |
|---|---|
| backend | `main` |
| cms | `main` |
| web | `master` |

`ref` kosong, `main`, atau `master` berarti "default branch" dan diterjemahkan per repo, jadi `service: all`
dengan `ref: main` tetap membuild web dari `master`. Ref lain (branch fitur, tag, SHA) dipakai apa adanya —
untuk ref seperti ini sebaiknya pilih satu service saja, karena belum tentu ada di ketiga repo.

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

## GitHub Actions: Deploy

Workflow [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) SSH ke VPS lalu menjalankan
`git pull` → `docker compose pull` → `docker compose up -d` → cek `/health` backend.
Sebelum deploy ia membatalkan proses kalau: `DEPLOY_WORK_DIR` kosong/tidak ada, folder itu bukan clone repo ini
(mis. tertukar dengan folder NuSewa), repo tidak di branch `main`, network `traefik_public` tidak ada,
atau `.env` belum dibuat / masih berisi placeholder.

Cara menjalankan:

- **Build + deploy sekaligus:** Actions → *Build & Push Images* → centang **deploy**
  (hanya untuk `ref` kosong/`main`/`master`, karena yang di-deploy tag `latest`).
- **Deploy saja / rollback:** Actions → *Deploy Ponpes* → isi `image_tag` (`latest` atau SHA pendek dari summary build).

**Secrets tambahan** (bisa sama dengan yang dipakai repo NuSewa, kecuali `DEPLOY_WORK_DIR`):

| Nama | Isi |
|---|---|
| `DEPLOY_HOST` | IP/hostname VPS |
| `DEPLOY_USER` | User SSH |
| `DEPLOY_SSH_KEY` | Private key SSH (tanpa passphrase) |
| `DEPLOY_SSH_PORT` | Port SSH (opsional, default `22`) |
| `DEPLOY_WORK_DIR` | Path absolut folder clone repo ini di VPS — folder yang langsung berisi `.git`, `docker-compose.yml`, dan `.env` (mis. `/opt/ponpes-infrastructure`). **Beda** dengan folder NuSewa. Repo harus di branch `main`. |
| `GHCR_READ_TOKEN` | Opsional. PAT `read:packages` kalau image GHCR private dan VPS belum `docker login ghcr.io` |

Job deploy memakai GitHub environment `production` (dibuat otomatis saat pertama jalan). Tambahkan
*required reviewers* di Settings → Environments kalau deploy perlu persetujuan.

### Trigger dari repo aplikasi (opsional)

```bash
curl -X POST \
  -H "Authorization: Bearer <PAT dengan akses Actions/Contents write ke repo infra>" \
  https://api.github.com/repos/<owner>/ponpes-mardhotullah-infrastructure/dispatches \
  -d '{"event_type":"build","client_payload":{"service":"backend","ref":"main"}}'
```

## Deploy dengan Docker Compose

| URL | Service |
|---|---|
| https://mardhotullah.com | ponpes-web (`www.` di-redirect ke sini) |
| https://cms.mardhotullah.com | ponpes-cms |
| https://api.mardhotullah.com | ponpes-backend |

### Berbagi VPS dengan NuSewa

Stack ini **tidak menjalankan Traefik sendiri**. Port 80/443 di VPS sudah dipakai `nusewa_traefik`, jadi
ponpes menumpang Traefik tersebut lewat network eksternal `traefik_public` dan certresolver `letsencrypt`
miliknya. Stack NuSewa harus jalan lebih dulu.

Supaya tidak bentrok:

- Semua service diberi nama `ponpes-*`. Container di `traefik_public` saling bisa resolve by name, dan NuSewa
  sudah memakai `backend`, `cms`, `web`, `minio` — tanpa prefix, `backend` dari container ponpes bisa
  mengarah ke backend NuSewa.
- Semua router/service/middleware Traefik diberi prefix `ponpes-`. Nama yang sama di dua stack membuat
  Traefik menonaktifkan router keduanya (situs NuSewa ikut mati).
- Project `ponpes`, network `ponpes_internal`, volume `ponpes_*`. Console MinIO di `127.0.0.1:9011`.

### Langkah (setup pertama di VPS)

1. Clone repo ini ke folder tersendiri (path ini = secret `DEPLOY_WORK_DIR`):
   `git clone https://github.com/amarkusuma/ponpes-mardhotullah-infrastructure.git /opt/ponpes-infrastructure`
2. Arahkan DNS **A record** `mardhotullah.com`, `www`, `cms`, `api` ke IP VPS.
3. Pastikan stack NuSewa jalan (`docker network inspect traefik_public`).
4. Buat `.env` lalu jalankan (selanjutnya cukup lewat workflow Deploy):

```bash
cp .env.example .env
# ganti SEMUA nilai "ganti-*" dengan nilai acak, contoh:
sed -i \
  -e "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=$(openssl rand -hex 24)/" \
  -e "s/^JWT_SECRET=.*/JWT_SECRET=$(openssl rand -hex 32)/" \
  -e "s/^MINIO_ACCESS_KEY=.*/MINIO_ACCESS_KEY=$(openssl rand -hex 10)/" \
  -e "s/^MINIO_SECRET_KEY=.*/MINIO_SECRET_KEY=$(openssl rand -hex 24)/" .env
grep "^[A-Z].*ganti-" .env && echo "MASIH ADA PLACEHOLDER" || echo OK
echo <PAT read:packages> | docker login ghcr.io -u <owner> --password-stdin   # jika package private
docker compose pull
docker compose up -d
```

Backend menjalankan migrasi & seeder otomatis saat start. Bucket MinIO dibuat otomatis.
Console MinIO (tidak dibuka ke internet): `ssh -L 9011:127.0.0.1:9011 root@IP_VPS`, lalu buka http://localhost:9011.

> Ganti `POSTGRES_PASSWORD` **sebelum** `up` pertama kali. Postgres hanya membaca nilai ini saat volume
> `ponpes_postgres_data` masih kosong; mengubahnya belakangan butuh `ALTER USER` manual.
