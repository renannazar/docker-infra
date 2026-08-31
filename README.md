# Docker Infrastructure

Shared Docker infrastructure for local development.

Repository ini menyediakan service yang dapat digunakan bersama oleh beberapa project development, sehingga setiap project tidak perlu menjalankan database dan service infrastructure masing-masing.

## Services

| Service    | Image               |   Port | Keterangan                |
| ---------- | ------------------- | -----: | ------------------------- |
| MySQL      | `mysql:8.4`         | `3306` | Relational database       |
| phpMyAdmin | `phpmyadmin:latest` | `8080` | Web interface untuk MySQL |
| Redis      | `redis:7-alpine`    | `6379` | Cache & queue             |

Semua service berada dalam Docker network:

```text
dev-network
```

## Requirements

* Docker
* Docker Compose
* WSL2 (jika menggunakan Windows)

Pastikan Docker dapat digunakan dari WSL:

```bash
docker --version
docker compose version
```

## Directory Structure

Direkomendasikan menyimpan repository ini langsung di home directory WSL:

```text
~/
├── docker_infra/
│   ├── compose.yml
│   ├── .env
│   └── README.md
│
└── projects/
    ├── project-a/
    ├── project-b/
    └── project-c/
```

`docker_infra` bertanggung jawab terhadap shared infrastructure, sedangkan masing-masing project bertanggung jawab terhadap application container-nya sendiri.

Contoh:

```text
docker_infra
├── MySQL
├── Redis
└── phpMyAdmin

project-a
├── Nginx
├── PHP-FPM
└── Worker

project-b
└── Go Application
```

## Configuration

Buat file `.env` berdasarkan konfigurasi berikut:

```env
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=default_db
MYSQL_USER=developer
MYSQL_PASSWORD=secret

MYSQL_PORT=3306
PHPMYADMIN_PORT=8080
REDIS_PORT=6379
```

> **Catatan:** `.env` digunakan untuk local development. Jangan gunakan password contoh di atas untuk production. Kamu juga dapat menggunakan `cp .env.example .env`

## Start Services

Jalankan seluruh infrastructure:

```bash
docker compose up -d
```

Periksa status:

```bash
docker compose ps
```

Melihat log:

```bash
docker compose logs
```

Melihat log service tertentu:

```bash
docker compose logs mysql
docker compose logs redis
docker compose logs phpmyadmin
```

## Stop Services

Untuk menghentikan container tanpa menghapusnya:

```bash
docker compose stop
```

Untuk menjalankan kembali:

```bash
docker compose start
```

Karena environment ini menggunakan:

```yaml
restart: "no"
```

service tidak akan otomatis dijalankan kembali setelah Docker/WSL2 restart.

Untuk menjalankan kembali seluruh infrastructure:

```bash
docker compose up -d
```

## phpMyAdmin

Setelah service berjalan, buka:

```text
http://localhost:8080
```

Gunakan:

```text
Server   : mysql
Username : developer
Password : secret
```

Atau gunakan user `root`:

```text
Server   : mysql
Username : root
Password : root
```

phpMyAdmin terhubung ke MySQL melalui Docker network menggunakan hostname:

```text
mysql
```

Bukan:

```text
localhost
```

## MySQL Connection

Project yang berada di Docker network `dev-network` dapat mengakses MySQL menggunakan:

```text
Host     : mysql
Port     : 3306
Database : <database-name>
Username : developer
Password : secret
```

Contoh konfigurasi Laravel:

```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=k_cbt
DB_USERNAME=developer
DB_PASSWORD=secret
```

## Redis Connection

Project yang berada di `dev-network` dapat mengakses Redis menggunakan:

```text
Host : redis
Port : 6379
```

Contoh konfigurasi Laravel:

```env
REDIS_HOST=redis
REDIS_PORT=6379
```

## Connecting Other Docker Projects

Project lain dapat menggunakan MySQL atau Redis dari infrastructure ini dengan bergabung ke network:

```text
dev-network
```

Contoh pada `compose.yml` project:

```yaml
networks:
  dev-network:
    external: true
    name: dev-network
```

Kemudian service aplikasi dapat menggunakan:

```yaml
services:
  app:
    networks:
      - dev-network

networks:
  dev-network:
    external: true
    name: dev-network
```

Setelah `docker_infra` dijalankan, network akan tersedia:

```bash
docker network ls
```

Contoh:

```text
dev-network
```

## Data Persistence

MySQL dan Redis menggunakan Docker named volume:

```text
mysql_data
redis_data
```

Data tetap tersedia meskipun container dihentikan atau dihapus dengan:

```bash
docker compose down
```

### Important

Jangan menggunakan:

```bash
docker compose down -v
```

kecuali memang ingin menghapus seluruh data MySQL dan Redis.

Perbandingannya:

```bash
docker compose stop
```

Menghentikan container tanpa menghapusnya.

```bash
docker compose down
```

Menghapus container dan network Compose, tetapi volume tetap dipertahankan.

```bash
docker compose down -v
```

Menghapus container, network, dan volume.

**`docker compose down -v` akan menghapus database development.**

## Checking Docker Resources

Melihat container yang sedang berjalan:

```bash
docker ps
```

Melihat semua container:

```bash
docker ps -a
```

Melihat volume:

```bash
docker volume ls
```

Melihat network:

```bash
docker network ls
```

Melihat resource yang digunakan Docker:

```bash
docker system df
```

## Git

File `.env` sebaiknya **tidak di-commit** ke repository.

Tambahkan ke `.gitignore`:

```gitignore
.env
```

Simpan konfigurasi contoh sebagai:

```text
.env.example
```

Contoh:

```env
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=default_db
MYSQL_USER=developer
MYSQL_PASSWORD=secret

MYSQL_PORT=3306
PHPMYADMIN_PORT=8080
REDIS_PORT=6379
```

Dengan demikian repository dapat dibagikan tanpa memasukkan konfigurasi lokal atau secret.

## Development Architecture

```text
                    dev-network
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
    MySQL              Redis          phpMyAdmin
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
           Project A             Project B
          PHP / Go / etc.       PHP / Go / etc.
```

Infrastructure dikelola secara terpisah dari application container agar database, cache, dan service pendukung dapat digunakan oleh beberapa project development.

## Production

Repository ini ditujukan untuk **local development**.

Production sebaiknya menggunakan infrastructure yang sesuai kebutuhan deployment, misalnya:

* Managed MySQL/PostgreSQL
* Managed Redis
* Database pada server terpisah
* Dedicated database server
* Containerized infrastructure dengan lifecycle terpisah

Jangan menggunakan konfigurasi password dan deployment pada repository ini secara langsung untuk production.
