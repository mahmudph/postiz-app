# Deploy Postiz di VPS (Traefik)

Panduan ini khusus untuk VPS yang sudah menjalankan Traefik bersama
(`/home/mahmud/server/treafik`, network `traefik`, certresolver `letsencrypt`).

## 1. Prasyarat di host

```bash
docker network create traefik   # sudah ada di VPS ini
```

DNS: buat record **A** `postiz.mahmudph.my.id` → IP VPS, proxy Cloudflare **ON**.
Traefik baru bisa menerbitkan sertifikat Let's Encrypt setelah record ini ada.

## 2. Siapkan `.env`

```bash
cp .env.prod.example .env
# isi POSTIZ_DB_PASSWORD, TEMPORAL_DB_PASSWORD, JWT_SECRET
#   openssl rand -hex 24   (password DB)
#   openssl rand -hex 32   (JWT_SECRET)
chmod 600 .env
```

## 3. Jalankan

```bash
docker compose -f docker-compose.prod.yml up -d
docker compose -f docker-compose.prod.yml ps
docker logs -f postiz
```

Tidak ada port yang dipublikasikan ke host. Satu-satunya jalan masuk adalah
Traefik lewat network `traefik`.

## 4. Setelah container sehat

1. Buka `https://postiz.mahmudph.my.id`, daftar akun pertama (jadi admin).
2. Set `DISABLE_REGISTRATION=true` di `.env`, lalu:
   ```bash
   docker compose -f docker-compose.prod.yml up -d postiz
   ```
3. Isi kredensial OAuth platform yang mau dipakai (X, LinkedIn, dst.) di `.env`,
   lalu recreate container yang sama.

## 5. Update

```bash
git pull
docker compose -f docker-compose.prod.yml pull postiz
docker compose -f docker-compose.prod.yml up -d
```

Image dibangun oleh GitHub Actions (`.github/workflows/build-containers.yml`)
dan dipublikasikan ke `ghcr.io/mahmudph/postiz-app:latest` setiap kali workflow
di-dispatch atau ada tag baru.

## Catatan operasional

- **RAM**: stack ini (postiz + 2 postgres + redis + temporal) butuh ~1.5 GB.
  VPS ini 3.8 GB dan sudah menjalankan bahasa-sumsel + model-service, jadi
  **swap 2 GB wajib ada** (`/swapfile`). Tanpa swap, container bisa OOM-killed.
- **Elasticsearch dimatikan**. Temporal memakai standard visibility di
  postgres. Karena itu `SKIP_ADD_CUSTOM_SEARCH_ATTRIBUTES=true` wajib: sample
  attribute bawaan Temporal bertipe Text dan memakan jatah 3 Text per
  namespace, sehingga Postiz gagal menambah `organizationId`/`postId`.
- **Upload**: Cloudflare membatasi body request 100 MB (free plan), sementara
  nginx Postiz mengizinkan 2 GB. Upload video besar akan ditolak Cloudflare.
- **Log Traefik**: stack Traefik menulis access log + debug log ke file di
  dalam container dan tidak dirotasi. Pernah mencapai ~900 MB. Cek berkala:
  ```bash
  docker exec traefik_reverse_proxy sh -c ': > /etc/traefik/access.log; : > /etc/traefik/traefik.log'
  ```
