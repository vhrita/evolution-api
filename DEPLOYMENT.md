# Guia de Implantação Evolution API

## 1. Desenvolvimento Local

### 1.1 Preparar Ambiente
```bash
# Clone o repositório
git clone https://github.com/EvolutionAPI/evolution-api.git
cd evolution-api

# O arquivo .env já está configurado para uso com docker-compose.dev.yaml
# Se necessário, ajuste as credenciais de banco e Redis
```

### 1.2 Teste Local Completo (API + Redis + PostgreSQL)
```bash
# Inicie todos os serviços (API, Redis, PostgreSQL)
docker-compose -f docker-compose.dev.yaml up --build

# A API estará disponível em:
# - http://localhost:8080
# - Manager UI: http://localhost:8080/manager
```

### 1.3 Desenvolvimento com Hot Reload (Opcional)
```bash
# Suba apenas Redis e PostgreSQL
docker-compose -f docker-compose.dev.yaml up -d redis evolution-postgres

# Instale as dependências
npm install

# Inicie o servidor de desenvolvimento
npm run dev:server
```

## 2. Build da Imagem Docker de Produção

### 2.1 Build Local
```bash
# Substitua SEU_REGISTRY pelo seu registry:
# - Docker Hub: seu-usuario/evolution-api
# - GitHub CR: ghcr.io/seu-usuario/evolution-api
# - Registry Privado: seu-registry.com/evolution-api

# Build da imagem
docker build -t SEU_REGISTRY/evolution-api:prod .

# Exemplo Docker Hub:
# docker build -t vitor/evolution-api:prod .

# Exemplo GitHub Container Registry:
# docker build -t ghcr.io/vitor/evolution-api:prod .
```

### 2.2 Push para Registry
```bash
# Login no registry (se necessário)
docker login
# ou para GHCR: docker login ghcr.io

# Push da imagem
docker push SEU_REGISTRY/evolution-api:prod
```

## 3. Deployment no CapRover

### 3.1 Pré-requisitos
Certifique-se de ter os seguintes serviços rodando na sua VPS:
- Redis
- PostgreSQL (ou MySQL)
- RabbitMQ (opcional, se usar eventos)

### 3.2 Primeira Configuração (Nova App)
1. Acesse o dashboard do CapRover
2. Crie uma nova app (ex: `evolution-api`)
3. Configure as variáveis de ambiente:
   ```bash
   # Básico
   SERVER_URL=https://sua-api.dominio.com
   AUTHENTICATION_API_KEY=429683C4C977415CAAFCCE10F7D57E11

   # Database
   DATABASE_PROVIDER=postgresql
   DATABASE_CONNECTION_URI=postgresql://user:pass@postgres-host:5432/evolution?schema=evolution_api

   # Cache
   CACHE_REDIS_ENABLED=true
   CACHE_REDIS_URI=redis://redis-host:6379/6

   # Demais variáveis conforme necessário
   ```
4. Configure volumes para persistência:
   - `/evolution/instances` → Para dados das instâncias

### 3.3 Deploy (Atualização de Imagem)
1. Vá até sua app existente no CapRover
2. Na aba **Deployment**, seção **Method 3: Deploy from ImageRegistry**
3. Cole o nome da imagem:
   ```
   SEU_REGISTRY/evolution-api:prod
   ```
4. Clique em **Deploy Now**

**Importante**: As variáveis de ambiente existentes serão mantidas!

### 3.4 Verificação
```bash
# Health check
curl https://sua-api.dominio.com/manager

# Listar instâncias (usando sua API key)
curl -H "apikey: 429683C4C977415CAAFCCE10F7D57E11" \
  https://sua-api.dominio.com/instance/fetchInstances

# Verificar logs no CapRover
caprover logs -a evolution-api
```

## 4. Atualização do Baileys

### 4.1 Atualização Local
```bash
# Atualize o Baileys para a versão mais recente
npm update @whiskeysockets/baileys

# Verifique a versão instalada
npm list @whiskeysockets/baileys
```

### 4.2 Teste Local
```bash
# Teste com docker-compose
docker-compose -f docker-compose.dev.yaml up --build

# Ou teste com npm
npm run dev:server

# Crie uma instância de teste e valide:
# - Conexão via QR Code
# - Envio de mensagens
# - Recebimento de mensagens
```

### 4.3 Build e Push da Nova Imagem
```bash
# Build com tag específica para Baileys
# Use versionamento semântico ou data
docker build -t SEU_REGISTRY/evolution-api:baileys-v2.3.6 .

# Exemplos de tags:
# - baileys-v2.3.6
# - baileys-2024-11-01
# - baileys-latest

# Push para registry
docker push SEU_REGISTRY/evolution-api:baileys-v2.3.6
```

### 4.4 Deploy da Nova Versão no CapRover
1. No CapRover, vá até sua app `evolution-api`
2. Deploy via imagem:
   ```
   SEU_REGISTRY/evolution-api:baileys-v2.3.6
   ```
3. Monitore os logs em tempo real
4. Teste funcionalidades críticas

### 4.5 Rollback (Se Necessário)
Em caso de problemas, volte para a versão anterior:
```
SEU_REGISTRY/evolution-api:prod
```

## 5. Estratégia de Versionamento

Recomendamos manter múltiplas tags para facilitar rollback:

```bash
# Tag de produção estável
SEU_REGISTRY/evolution-api:prod

# Tag com versão específica do Baileys
SEU_REGISTRY/evolution-api:baileys-v2.3.6

# Tag com data de deploy
SEU_REGISTRY/evolution-api:2024-11-01

# Tag latest (sempre a mais recente)
SEU_REGISTRY/evolution-api:latest
```

Exemplo de build com múltiplas tags:
```bash
docker build -t SEU_REGISTRY/evolution-api:prod \
             -t SEU_REGISTRY/evolution-api:baileys-v2.3.6 \
             -t SEU_REGISTRY/evolution-api:latest .

docker push SEU_REGISTRY/evolution-api:prod
docker push SEU_REGISTRY/evolution-api:baileys-v2.3.6
docker push SEU_REGISTRY/evolution-api:latest
```

## 6. Verificações de Saúde

### 6.1 Endpoints de Teste
- `GET /manager` - Interface de gerenciamento
- `GET /instance/fetchInstances` - Lista de instâncias
- `POST /instance/create` - Criar nova instância

### 6.2 Logs e Monitoramento
```bash
# Logs do container (Docker local)
docker logs -f evolution_api

# Logs no CapRover
caprover logs -a evolution-api

# Status das instâncias
curl -H "apikey: 429683C4C977415CAAFCCE10F7D57E11" \
  http://localhost:8080/instance/fetchInstances
```

## 7. Troubleshooting

### 7.1 Problemas Comuns

**Container não inicia:**
```bash
# Verificar logs
docker logs evolution_api

# Verificar conexão com banco
docker exec evolution_api nc -zv evolution-postgres 5432

# Verificar conexão com Redis
docker exec evolution_api nc -zv evolution_redis 6379
```

**Erro de migração de banco:**
```bash
# Executar migrations manualmente
docker exec evolution_api npm run db:deploy
```

**Instâncias não conectam:**
- Verificar se o volume `/evolution/instances` está persistido
- Verificar logs do Baileys: `LOG_BAILEYS=debug`
- Verificar firewall e portas abertas

### 7.2 Limpeza de Recursos

```bash
# Parar todos os containers
docker-compose -f docker-compose.dev.yaml down

# Limpar volumes (CUIDADO: apaga dados!)
docker-compose -f docker-compose.dev.yaml down -v

# Limpar imagens antigas
docker image prune -a
```
