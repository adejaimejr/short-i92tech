# Shlink - Encurtador de Links | short.i92tecnologia.com.br

Encurtador de links auto-hospedado usando [Shlink](https://shlink.io), pronto para deploy em produção no Dokploy com Docker Swarm e Traefik.

## Estrutura do Projeto

```
/
├── docker-compose.yml   # Stack do Shlink para Docker Swarm
├── .env.example         # Template de variáveis de ambiente
├── .gitignore           # Arquivos ignorados pelo Git
└── README.md            # Este arquivo
```

## Como Configurar o .env

```bash
cp .env.example .env
```

Edite o `.env` com seus valores reais:

| Variável | Descrição | Exemplo |
|---|---|---|
| `TZ` | Timezone | `America/Sao_Paulo` |
| `SHLINK_DEFAULT_DOMAIN` | Domínio público | `short.i92tecnologia.com.br` |
| `SHLINK_IS_HTTPS_ENABLED` | HTTPS ativo | `true` |
| `SHLINK_TRUSTED_PROXIES` | IPs confiáveis do proxy | `*` |
| `SHLINK_DB_DRIVER` | Driver do banco | `postgres` |
| `SHLINK_DB_HOST` | Host do PostgreSQL | `seu-host` |
| `SHLINK_DB_PORT` | Porta do PostgreSQL | `5432` |
| `SHLINK_DB_NAME` | Nome do banco | `shlink` |
| `SHLINK_DB_USER` | Usuário do banco | `seu-usuario` |
| `SHLINK_DB_PASSWORD` | Senha do banco | `sua-senha` |
| `SHLINK_INITIAL_API_KEY` | API Key (gerada no 1o boot) | resultado de `openssl rand -hex 32` |
| `SHLINK_GEOLITE_LICENSE_KEY` | Licença GeoLite2 (opcional) | deixe vazio se não usar |

**Gerar uma API Key segura:**
```bash
openssl rand -hex 32
```

## Como Subir no Dokploy

### Passo a Passo para Publicar no GitHub

1. Acesse o repositório: https://github.com/adejaimejr/short-i92tech

2. Se ainda não inicializou o Git local:
```bash
cd /caminho/do/projeto
git init
git remote add origin git@github.com:adejaimejr/short-i92tech.git
```

3. Faça o push:
```bash
git add .
git commit -m "feat: stack Shlink pronta para produção no Dokploy"
git branch -M main
git push -u origin main
```

### Passo a Passo para Subir no Dokploy

1. No painel do Dokploy, crie um novo **Application**
2. Selecione **Docker Compose** como tipo de deploy
3. Aponte para o repositório GitHub: `adejaimejr/short-i92tech`
4. Em **Environment Variables**, adicione todas as variáveis do `.env` com os valores reais
5. Faça o **Deploy**
6. Após o deploy, vá em **Domains** no Dokploy e adicione:
   - Domínio: `short.i92tecnologia.com.br`
   - Porta do container: `8080`
   - HTTPS habilitado com certResolver `letsencrypt`
7. O Dokploy vai gerar automaticamente o arquivo de roteamento do Traefik

**Importante sobre o Traefik no Dokploy:**
O Dokploy usa o **file provider** do Traefik (não labels do Swarm). O arquivo de roteamento é gerado automaticamente em `/root/swarm/stacks/applications/dokploy/data/traefik/dynamic/` ao configurar o domínio no UI. O Traefik recarrega automaticamente.

Se precisar verificar/corrigir o arquivo gerado, o formato correto é:
```yaml
http:
  routers:
    shlink-router:
      rule: "Host(`short.i92tecnologia.com.br`)"
      service: shlink-service
      middlewares: []
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt
  services:
    shlink-service:
      loadBalancer:
        servers:
          - url: http://<nome-do-servico-no-swarm>:8080
        passHostHeader: true
```

## Como Validar se o Shlink Iniciou

```bash
# Ver logs do serviço
docker service logs <nome-do-servico> --tail 50

# Testar health check
docker exec $(docker ps -q -f name=shlink) wget -qO- http://localhost:8080/rest/health

# Resposta esperada:
# {"status":"pass","version":"4.x.x","links":{"about":"https://shlink.io","project":"https://github.com/shlinkio/shlink"}}
```

## Como Acessar o Serviço

Após o deploy e configuração do domínio:
- URL principal: `https://short.i92tecnologia.com.br`
- Health check: `https://short.i92tecnologia.com.br/rest/health`

## API Key

A `SHLINK_INITIAL_API_KEY` definida no `.env` é criada automaticamente no primeiro boot. Use-a para todas as chamadas à API.

Se precisar gerar uma nova API key depois:
```bash
docker exec -it $(docker ps -q -f name=shlink) shlink api-key:generate
```

Para listar as API keys existentes:
```bash
docker exec -it $(docker ps -q -f name=shlink) shlink api-key:list
```

## Exemplo de Uso da API

### Criar uma Short URL

**Endpoint:** `POST https://short.i92tecnologia.com.br/rest/v3/short-urls`

**Headers:**
```
X-Api-Key: sua-api-key-aqui
Content-Type: application/json
```

**Body (JSON):**
```json
{
  "longUrl": "https://i92tecnologia.com.br/minha-pagina-muito-longa",
  "customSlug": "minha-pagina",
  "findIfExists": true
}
```

**Resposta:**
```json
{
  "shortCode": "minha-pagina",
  "shortUrl": "https://short.i92tecnologia.com.br/minha-pagina",
  "longUrl": "https://i92tecnologia.com.br/minha-pagina-muito-longa",
  "dateCreated": "2026-03-14T12:00:00+00:00",
  "tags": [],
  "meta": {
    "validSince": null,
    "validUntil": null,
    "maxVisits": null
  }
}
```

**Teste com curl:**
```bash
curl -X POST https://short.i92tecnologia.com.br/rest/v3/short-urls \
  -H "X-Api-Key: sua-api-key-aqui" \
  -H "Content-Type: application/json" \
  -d '{"longUrl": "https://i92tecnologia.com.br", "customSlug": "home"}'
```

### Listar Short URLs

```bash
curl -s https://short.i92tecnologia.com.br/rest/v3/short-urls \
  -H "X-Api-Key: sua-api-key-aqui" | python3 -m json.tool
```

## Integração com n8n

Para encurtar links automaticamente via n8n, use o nó **HTTP Request**:

| Campo | Valor |
|---|---|
| **Method** | `POST` |
| **URL** | `https://short.i92tecnologia.com.br/rest/v3/short-urls` |
| **Authentication** | Header Auth |
| **Header Name** | `X-Api-Key` |
| **Header Value** | `sua-api-key-aqui` |
| **Body Content Type** | JSON |
| **Body** | `{"longUrl": "{{ $json.url }}", "findIfExists": true}` |

O campo `{{ $json.url }}` assume que o nó anterior no workflow entrega a URL longa em `url`. Ajuste conforme seu workflow.

**Dica:** Use `findIfExists: true` para evitar duplicatas — se a URL longa já foi encurtada, retorna a short URL existente.

## Cuidados de Produção

- **Nunca commite o `.env`** — ele contém senhas e API keys
- **TRUSTED_PROXIES=*** é seguro quando o Shlink não está exposto diretamente na internet (está atrás do Traefik)
- **Backup do banco**: o Shlink armazena tudo no PostgreSQL — garanta que seu banco tem backup
- **HTTPS**: o Traefik gerencia os certificados via Let's Encrypt automaticamente
- **Cloudflare**: se usar proxy Cloudflare (nuvem laranja), configure o SSL mode como **Full**
- **Redeploy**: após redeploy no Dokploy, verifique se o arquivo de roteamento do Traefik mantém a porta `8080` e o `certResolver: letsencrypt`

## Troubleshooting

### Shlink não inicia
```bash
# Ver logs detalhados
docker service logs <nome-servico> --tail 100 --no-trunc

# Verificar se o banco está acessível
docker exec $(docker ps -q -f name=shlink) sh -c "wget -qO- http://localhost:8080/rest/health"
```

### Domínio não responde
```bash
# Verificar se o Traefik enxerga o router
docker exec $(docker ps -q -f name=traefik) wget -qO- http://localhost:8080/api/http/routers \
  | python3 -m json.tool | grep -A5 "shlink"

# Verificar erros no Traefik
docker exec $(docker ps -q -f name=traefik) wget -qO- http://localhost:8080/api/http/routers \
  | python3 -m json.tool | grep -E "error|disabled"

# Verificar se o serviço está na rede correta
docker service inspect <nome-servico> | jq '.[0].Spec.TaskTemplate.Networks'

# Testar porta interna diretamente
docker exec $(docker ps -q -f name=traefik) wget -qO- http://<nome-servico>:8080
```

### Certificado SSL com erro
- Se usar Cloudflare com proxy (nuvem laranja): SSL mode deve ser **Full**
- Se usar DNS only: Let's Encrypt resolve automaticamente
- Verificar logs do Traefik: `docker service logs traefik_traefik --tail 50`

### Erro de conexão com PostgreSQL
- Verifique se `SHLINK_DB_HOST` está correto e acessível de dentro da rede Docker
- Se o PostgreSQL está em outro serviço Swarm, use o nome do serviço como host
- Verifique se o banco `shlink` existe: `psql -h host -U user -d shlink -c "SELECT 1"`

### API retorna 401
- Verifique se a API key está correta
- Liste as keys: `docker exec -it $(docker ps -q -f name=shlink) shlink api-key:list`

## Observações sobre Traefik

- O Dokploy usa **file provider** do Traefik, não labels do Swarm
- As labels no `docker-compose.yml` servem como documentação — o roteamento real é via arquivo `.yml` gerado pelo Dokploy
- Sintaxe correta para múltiplos hosts no Traefik v3: `Host(`a.com`) || Host(`www.a.com`)`
- **Não usar**: `Host(`a.com`, `www.a.com`)` — essa é sintaxe v2
- O Traefik recarrega os arquivos automaticamente (file watch)
- Após redeploy, os arquivos são sobrescritos — confirme porta e certResolver

## Observações sobre PostgreSQL Externo

- Esta stack **não cria** um serviço de PostgreSQL
- O Shlink se conecta ao PostgreSQL já existente no ambiente via variáveis de ambiente
- Certifique-se de que o banco de dados `shlink` existe antes do primeiro deploy
- O Shlink cria as tabelas automaticamente no primeiro boot
- Se o host do PostgreSQL é um serviço Swarm, use o nome do serviço (ex: `postgres`) como `SHLINK_DB_HOST`

## Checklist Final de Produção

- [ ] `.env` criado com valores reais (não commitar!)
- [ ] `SHLINK_INITIAL_API_KEY` gerada com `openssl rand -hex 32`
- [ ] Banco de dados `shlink` criado no PostgreSQL
- [ ] Repositório no GitHub atualizado
- [ ] Deploy feito no Dokploy
- [ ] Domínio `short.i92tecnologia.com.br` configurado no Dokploy (porta 8080)
- [ ] DNS apontando para o servidor (A record ou CNAME)
- [ ] Health check respondendo: `https://short.i92tecnologia.com.br/rest/health`
- [ ] API key funcionando: testar criação de short URL
- [ ] Cloudflare SSL mode = Full (se usar proxy Cloudflare)
- [ ] Backup do PostgreSQL configurado
