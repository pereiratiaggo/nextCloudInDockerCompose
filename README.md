# 🚀 Nextcloud + Docker + Nginx Proxy Manager (com suporte a HTTPS e clientes oficiais)

Este guia descreve como configurar o **Nextcloud** usando **Docker Compose**, com banco de dados MariaDB, Redis e proxy reverso **Nginx Proxy Manager** para gerenciar certificados SSL (Let's Encrypt).  
Inclui correções para o cliente oficial do Nextcloud (desktop e mobile), evitando erros de HTTPS.

---

## ✅ Checklist de Configuração

### 1. Estrutura do `docker-compose.yml`
Crie um arquivo `docker-compose.yml` com os serviços:

```yaml
version: "3.8"

services:
  db:
    image: mariadb:lts
    restart: always
    command: --transaction-isolation=READ-COMMITTED
    volumes:
      - db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE:-nextcloud}
      - MYSQL_USER=${MYSQL_USER:-nextcloud}

  redis:
    image: redis:alpine
    restart: always

  app:
    image: nextcloud
    restart: always
    expose:
      - "80"                # exposto apenas para a rede interna
    depends_on:
      - redis
      - db
    volumes:
      - nextcloud:/var/www/html
      - ./data:/var/www/html/data
    environment:
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_HOST=db

  proxy:
    image: jc21/nginx-proxy-manager:latest
    restart: always
    ports:
      - "80:80"
      - "81:81"   # painel de admin
      - "443:443"
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt

volumes:
  db:
  nextcloud:
  npm_data:
  npm_letsencrypt:
```

---

### 2. Subir containers
```bash
docker compose up -d
```

---

### 3. Configurar Proxy Reverso (Nginx Proxy Manager)
1. Acesse o painel: `http://SEU_IP:81`  
2. Login inicial:
   - **E-mail:** `admin@example.com`  
   - **Senha:** `changeme`  
3. Troque e-mail e senha.  
4. Crie um **Proxy Host**:
   - **Domain:** `nextcloud.seudominio.com`  
   - **Forward Hostname/IP:** `app`  
   - **Forward Port:** `80`  
5. Ative as opções:
   - ✅ Force SSL  
   - ✅ HTTP/2 Support  
   - ✅ Request SSL Certificate (Let's Encrypt)

---

### 4. Ajustar configuração do Nextcloud
Edite o arquivo `config/config.php` dentro do container:

```bash
docker exec -it nextcloud-app bash
nano /var/www/html/config/config.php
```

Adicione/ajuste:
```php
'trusted_domains' =>
  array (
    0 => 'localhost',
    1 => 'nextcloud.seudominio.com',
  ),
'overwrite.cli.url' => 'https://nextcloud.seudominio.com',
'overwriteprotocol' => 'https',
```

Reinicie o container:
```bash
docker restart nextcloud-app
```

---

### 5. Testes de Verificação
- **Navegador**:  
  Acesse `https://nextcloud.seudominio.com` → deve abrir com 🔒 válido.  

- **WebDAV** (linha de comando):  
  ```bash
  curl -u USUARIO:SENHA https://nextcloud.seudominio.com/remote.php/dav/files/USUARIO/
  ```
  → deve listar arquivos.  

- **Cliente oficial Nextcloud (PC/Mobile)**:  
  Configure usando exatamente:
  ```
  https://nextcloud.seudominio.com
  ```

---

## 📌 Observações
- O cliente Nextcloud **não aceita certificados self-signed** → precisa ser Let's Encrypt ou outro válido.  
- O parâmetro `'overwriteprotocol' => 'https'` é essencial para evitar erros de redirecionamento no login do cliente.  
- O domínio deve estar corretamente apontado para o servidor (via DNS).  

---
