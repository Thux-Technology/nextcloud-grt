# nextcloud-grt

Deploy de Nextcloud em Docker com suporte a external storage — stack de produção gerida pela [Thux Tech](https://thux.tech).

---

## Stack

- **Nextcloud** (Apache) — aplicação principal
- **MariaDB 11.4** — banco de dados
- **Nginx** (host) — reverse proxy com SSL
- **Certbot** — certificado Let's Encrypt via DNS-01 (Cloudflare)

---

## Estrutura

```
nextcloud-grt/
├── docker-compose.yml        # definição dos serviços
├── .env.example              # template de variáveis (sem valores reais)
├── .gitignore
└── nginx/
    └── grt.thux.tech.conf    # config do reverse proxy com SSL
```

Diretórios no host (não versionados):

```
/opt/nextcloud/
├── config/     # config.php — nunca versionar
├── data/       # datadirectory dos utilizadores
├── db/         # dados do MariaDB
└── external/   # external storage montado como /mnt/grt no container
```

---

## Deploy

### 1. Pré-requisitos

- Docker + Docker Compose Plugin
- Nginx instalado no host
- Certbot com plugin `python3-certbot-dns-cloudflare` (via dnf/apt)

### 2. Clonar e configurar

```bash
git clone git@github.com:thux-tech/nextcloud-grt.git /opt/nextcloud
cd /opt/nextcloud

cp .env.example .env
nano .env  # preencher com as credenciais reais
chmod 600 .env
```

### 3. Criar estrutura de diretórios

```bash
mkdir -p /opt/nextcloud/{config,data,db,nginx,external}
```

### 4. Subir os containers

```bash
docker compose up -d
docker compose logs -f app  # aguardar inicialização
```

### 5. Configurar SSL

```bash
mkdir -p /root/.secrets
cat > /root/.secrets/cloudflare.ini << 'EOF'
dns_cloudflare_api_token = <TOKEN>
EOF
chmod 600 /root/.secrets/cloudflare.ini

certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  -d <dominio> \
  --agree-tos -m <email>
```

### 6. Activar Nginx

```bash
cp nginx/grt.thux.tech.conf /etc/nginx/conf.d/
nginx -t && systemctl reload nginx
```

---

## Operações comuns

```bash
# status
docker compose ps
docker exec -u www-data nextcloud-app php occ status

# modo manutenção
docker exec -u www-data nextcloud-app php occ maintenance:mode --on
docker exec -u www-data nextcloud-app php occ maintenance:mode --off

# logs
docker compose logs app -f

# backup do banco
mariadb-dump -u <user> -p --single-transaction <database> > backup.sql

# atualizar versão
# 1. editar docker-compose.yml: nextcloud:X.X.X-apache
docker compose pull app
docker compose up -d app
docker exec -u www-data nextcloud-app php occ upgrade
docker exec -u www-data nextcloud-app php occ maintenance:repair
```

---

## Variáveis de ambiente

Ver `.env.example` para a lista completa. O arquivo `.env` com valores reais **nunca** deve ser commitado.

---

## Notas de segurança

- Container roda como `www-data` (uid 33) — ficheiros em `/opt/nextcloud/data/`, `/opt/nextcloud/config/` e `/opt/nextcloud/external/` devem pertencer ao uid 33
- `no-new-privileges: true` e `tmpfs` em `/tmp` activados em ambos os containers
- Nginx exposto nas portas 80/443 do host; container da app apenas em `127.0.0.1:8080`
- `config/` excluído do repositório — contém credenciais e chaves secretas

---

## Autor

**Hudson Bento** · [Thux Tech](https://thux.tech) · [LinkedIn](https://www.linkedin.com/in/hudsonbento/)
