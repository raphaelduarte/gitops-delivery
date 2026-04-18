# gitops-delivery

Repo centralizado de delivery — composes que o Portainer puxa pra deployar as apps pessoais.

Inspirado no padrão do `gitops-delivery` do PCP (Marketplace), mas com **Docker Compose** em vez de Helm/K8s, e **Portainer** em vez de ArgoCD.

## Estrutura

```
applications/
├── dev/
│   ├── recuperai/docker-compose.yml
│   ├── linkshort/docker-compose.yml      (futuro)
│   └── ...
├── hom/
│   └── ...                               (futuro)
└── prod/
    └── ...                               (futuro)
```

Cada pasta `<env>/<app>/` contém UM `docker-compose.yml` com toda a stack daquela app naquele ambiente.

## Convenções

### Imagens
Sempre vêm de **GHCR**, buildadas no CI do repo do app. Nunca `build:` aqui — esse repo é só payload pronto.

```yaml
# Certo
image: ghcr.io/raphaelduarte/recuperai-api:latest

# Errado (esse repo não tem o código fonte)
build:
  context: ../../recuperai
```

### Networks
Toda app que precisa ser acessível externamente entra na network externa `easypanel-geral` (overlay Swarm gerenciado pelo Easypanel) — é onde o Traefik enxerga.

```yaml
networks:
  - default              # interna da stack (DB, redis, etc)
  - easypanel-geral      # exposta ao Traefik
```

### Roteamento
Via **labels Traefik** no compose. Easypanel não conhece essas stacks (foram criadas pelo Portainer), então a UI de Domínios dele não funciona aqui.

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.docker.network=easypanel-geral"
  - "traefik.http.routers.<app>.rule=Host(`<sub>.zepapagaio.com`)"
  - "traefik.http.routers.<app>.entrypoints=websecure"
  - "traefik.http.routers.<app>.tls.certresolver=letsencrypt"
  - "traefik.http.services.<app>.loadbalancer.server.port=<porta_interna>"
```

CNAME wildcard `*.zepapagaio.com` no Cloudflare resolve qualquer subdomínio automaticamente.

### Secrets
Setar no Portainer ao cadastrar a stack (aba **Environment variables**). Por enquanto manual — futura migração pra HashiCorp Vault (consumido via entrypoint script no container).

### Versionamento de imagens
- `:latest` em dev (deploy contínuo)
- `:<sha>` em prod (deploy explícito por push do delivery, não por push do código)

## Como cadastrar nova stack no Portainer

1. **Stacks → Add stack**
2. Nome: `<app>-<env>` (ex: `recuperai-dev`)
3. **Build method: Repository**
4. Repository URL: `git@github.com:raphaelduarte/gitops-delivery.git`
5. Repository reference: `refs/heads/main`
6. Compose path: `applications/<env>/<app>/docker-compose.yml`
7. **GitOps updates: ON** — Polling 1min ou Webhook
8. **Environment variables**: cola as vars que o compose espera
9. **Deploy the stack**

A partir daí, qualquer push neste repo (ou nova imagem com `:latest`) redeploya a stack.

## Pré-requisitos no Portainer

- **Registry credentials pro GHCR**: se o package no GHCR for privado, configurar em Registries → Add registry → GitHub → PAT com scope `read:packages`. Se for público, não precisa.

## Apps registradas

| App | Env | Subdomínio | Repo do código |
|---|---|---|---|
| recuperai-api | dev | recuperai-api-dev.zepapagaio.com | github.com/raphaelduarte/recuperai |
