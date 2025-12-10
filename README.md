# Módulo Auth — PortalWeb

Escopo: autenticação stateless com JWT (access/refresh), revogação por blacklist (JTI) e endpoints públicos de login/refresh, além de utilitário `/me`.

## Visão Geral

- Token Access: curto prazo, portador (Bearer), contém claims `sub` (email), `uid` (ID usuário), `role` e `jti`.
- Token Refresh: longo prazo, claim `typ=refresh`, também com `jti` exclusivo.
- Revogação: tabela `token_revogado` guarda `jti` já revogados (logout/comprometimento).

## DDL

```sql
CREATE TABLE IF NOT EXISTS token_revogado (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  jti VARCHAR(64) NOT NULL,
  revoked_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_token_revogado_jti (jti)
);
```

Observações:
- Índice único em `jti` garante idempotência nas revogações.
- Limpeza sugerida: job periódico removendo registros com `revoked_at < NOW() - INTERVAL 90 DAY` (ajuste conforme política).

## Endpoints

- POST `/api/v1/auth/login`
  - Body: `{ "email": string, "senha": string }`
  - 200: `{ token_type, access_token, access_expires_in_sec, refresh_token, refresh_expires_in_sec }`

- POST `/api/v1/auth/refresh`
  - Body: `{ "refresh_token": string }`
  - 200: mesmo payload de login, com novos access/refresh e novos JTIs.
  - 400/401: rejeita refresh sem `typ=refresh`, com `jti` inexistente, ou revogado.

- POST `/api/v1/auth/revoke`
  - Body: `{ "token": string }` (access ou refresh)
  - 204: revogação aceita (idempotente).

- GET `/api/v1/auth/me`
  - Header: `Authorization: Bearer <access>`
  - 200: `{ "email": string }`

## Claims do JWT

- `iss`: emissor (configurável, default `portalweb`).
- `sub`: email.
- `uid`: ID do usuário (Long).
- `role`: papel/autoridade principal.
- `typ`: presente apenas em refresh (`refresh`).
- `jti`: ID único por token (usado na blacklist).
- `iat`, `exp`: emissão e expiração.

## Configuração (application.properties)

- `app.auth.jwt.secret` — segredo HMAC (mín. 256-bit para HS256).
- `app.auth.jwt.issuer` — emissor (`portalweb` por padrão).
- `app.auth.jwt.access-ttl-sec` — TTL do access (default 900s).
- `app.auth.jwt.refresh-ttl-sec` — TTL do refresh (default 1209600s / 14 dias).

## Fluxos

1) Login: valida credenciais, emite par de tokens (ambos com `jti`).
2) Refresh: valida `typ=refresh`, confere `jti` contra `token_revogado` e emite novos tokens.
3) Revogação: extrai `jti` do token apresentado; grava em `token_revogado` se ainda não existir.

## Segurança e Boas Práticas

- Usar HTTPS sempre; nunca trafegar tokens em URL.
- Rotacionar `secret` sob governança, invalidando tokens quando necessário.
- Revogar refresh no logout; opcional: revogar access em incidentes.
- Considerar rate-limiting em `/login` e `/refresh`.

## Observabilidade

- Registrar tentativas de login inválidas (sem PII sensível) e revogações.
- Métricas: contagem de refresh válidos/negados, revogações/dia.

