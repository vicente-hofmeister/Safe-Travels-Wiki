# Capítulo 10 — Divergências e Pendências

Este capítulo consolida as diferenças entre o sistema planejado no TCC I e o que foi efetivamente implementado, registrando as decisões tomadas durante o desenvolvimento.

---

## 10.1 Divergências documentadas

### D1 — Arquitetura: AWS → Docker/Render

**Planejado:** API Gateway + AWS Lambda (SAM) + Kinesis Data Streams para processamento assíncrono de localização + RDS PostgreSQL.

**Implementado:** Node.js convencional em Docker Compose (desenvolvimento) e Render.com (produção), com inserção síncrona direta no PostgreSQL. Banco de dados local via Docker; em produção, Neon PostgreSQL.

**Motivo:** A conta AWS utilizada havia esgotado o período de elegibilidade ao free tier em uso anterior. Além disso, no período de desenvolvimento, a AWS alterou os termos do seu free tier, tornando necessário um replanejamento completo antes mesmo de iniciar a implementação. Optou-se por uma solução convencional para focar no desenvolvimento das funcionalidades do sistema.

**Impacto na implementação:** O processamento assíncrono via Kinesis foi eliminado. Toda localização é gravada diretamente no banco na mesma requisição HTTP. Para o volume de usuários do TCC, o comportamento é equivalente.

Arquivos legados mantidos como registro histórico: `apiHandler.ts`, `template.yaml`, `samconfig.toml`.

---

### D2 — Convenção de URLs: verbos → REST

**Planejado (Swagger):** URLs no estilo `/resource/verb` (ex: `/group/create`, `/group/leave/{id}`, `/group/remove-member/{group_id}/{user_id}`).

**Implementado:** REST canônico com verbos HTTP e recursos hierárquicos (ex: `POST /group`, `DELETE /group/:groupId/members/:userId`).

**Motivo:** A convenção REST é o padrão de mercado para APIs HTTP. O Swagger original usava um estilo que mistura recursos e verbos na URL, o que é considerado um anti-padrão.

---

### D3 — Login: `username` → `email` ✅ Resolvido

**Planejado (Swagger):** Login usando campo `username`.

**Implementado:** Login usando campo `email`.

**Decisão definitiva:** email é mais adequado como identificador de login — é único por definição no sistema e não pode ser trocado pelo usuário. O Swagger será atualizado para refletir essa decisão.

---

### D4 — Adição de membros: convite → adição direta

**Planejado (Swagger):** Fluxo de convite em dois passos — `POST /group/invite` seguido de `POST /group/respond-invite`.

**Implementado:** Adição direta pelo owner via `POST /group/:groupId/members`. Sem etapa de aceite pelo usuário convidado.

**Decisão:** Adição direta é o comportamento do MVP. O fluxo de convite adiciona complexidade (estado de convite pendente, notificações, expiração) que não é essencial para validar o conceito central do sistema.

---

### D5 — Sistema de follows e privacidade

**Planejado:** Tabela `follows` com tiers (`regular`, `close`, etc.) controlaria a granularidade de localização compartilhada. Campos `privacy_settings` em `users` e por grupo.

**Status atual:** Não implementado. A tabela `follows` não foi criada.

**Decisão:** O sistema de follows e privacidade granular não foi cortado formalmente, mas está indefinido para o MVP. O impacto é significativo (nova tabela, múltiplos módulos afetados), por isso não será implementado sem decisão explícita de escopo.

---

### D6 — Endpoints extras na implementação ✅ Resolvido

O Swagger original não previa vários endpoints que foram adicionados durante a implementação:

**Módulo de grupo:**
- `GET /group/:groupId` — consulta de grupo por ID (com lista de membros)
- `GET /group/user/:userId` — grupos de um usuário
- `DELETE /group/:groupId` — deleção de grupo

**Módulo de localização:**
- `GET /location/latest` — feed de localização para o mapa
- `GET /location/latest?userIds=...` — filtrado por lista de usuários
- `GET /location/id/:locationEventId` — busca por evento específico
- `GET /location/user/:userId` — última posição de um usuário

**Decisão:** O Swagger será atualizado para refletir a implementação real.

---

### D7 — Modelo de localização: campo inline → tabela dedicada

**Planejado (DBML):** Campo `location_data` nas tabelas `users`, `groups` e `trips`, armazenando a posição atual como atributo da entidade.

**Implementado:** Tabela separada `location_events` com histórico completo de posições, vinculada a grupos via `location_event_groups`.

**Motivo:** A remoção do Kinesis (D1) torna a localização um evento persistível diretamente no banco. Uma tabela dedicada permite manter histórico, indexar eficientemente e vincular o mesmo evento a múltiplos grupos.

---

## 10.2 Módulos pendentes

### Módulo `user` — stub

Apenas o health check (`GET /user/health`) está implementado. Os seguintes endpoints planejados ainda não existem:

| Endpoint | Observação |
|---|---|
| `GET /user/get-profile/:userId` | Perfil do usuário |
| Demais endpoints de follows | Dependem do sistema de follows (D5) |

### Módulo `trip` — stub

Apenas o health check (`GET /trip/health`) está implementado. A tabela `trips` não foi criada. Os endpoints planejados:

| Endpoint |
|---|
| `GET /trip/get-trip-summary/:tripId` |
| `GET /trip/get-trip-details/:tripId` |
| `GET /trip/list-trips/user/:userId` |
| `GET /trip/list-trips/group/:groupId` |
| `GET /trip/get-user-current-trip/:userId` |
| `GET /trip/get-group-current-trip/:groupId` |

A tabela `location_event_trips` foi criada no schema com FK apenas para `location_events` (a FK para `trips` será adicionada quando o módulo for implementado).

### Módulo `auth` — parcialmente implementado

| Endpoint | Status | Observação |
|---|---|---|
| `POST /auth/register` | ✅ | Implementado |
| `POST /auth/login` | ✅ | Implementado (usa email, não username — D3) |
| `POST /auth/refresh` | ❌ | Requer estado no servidor (refresh token store) |
| `POST /auth/logout` | ❌ | Requer revogação de token |
| `POST /auth/forgot-password` | ❌ | Requer serviço de email externo |
| `POST /auth/reset-password` | ❌ | — |
| `POST /auth/change-password` | ❌ | — |
| `POST /auth/verify-email` | ❌ | Requer serviço de notificação |
| `POST /auth/verify-phone` | ❌ | — |
| `POST /auth/register-device` | ❌ | — |

---

## 10.3 Pendências de UX no mobile

### Duplicata de localização ao abrir o mapa

A `MapScreen` chama `registerLocation` ao abrir. O background task pode registrar uma posição no mesmo momento, gerando dois eventos com `captured_at` idêntico ou muito próximo. A solução prevista é remover o `registerLocation` da `MapScreen` e delegar todo o envio ao background tracking.

### Controle do background tracking pelo usuário

Atualmente não há interface para o usuário ligar ou desligar o rastreamento em background — ele se inicia automaticamente ao fazer login e nunca é interrompido (exceto ao fechar o app). A tela de Home ou uma tela de Configurações deve receber um toggle para controle explícito pelo usuário.

---

## 10.4 Banco de dados — divergências em relação ao DBML original

| Tabela (DBML) | Status | Diferenças notáveis |
|---|---|---|
| `users` | ✅ Implementada | Faltam: `phone`, `profile_image`, `privacy_settings`. Adicionado: `password_hash`. `email` é coluna scalar (DBML previa array). |
| `follows` | ❌ Não existe | Sistema de follows/tiers não criado |
| `groups` | ✅ Implementada | Faltam: `profile_image`, `location_data` inline. Adicionado: `description` |
| `group_members` | ✅ Implementada | Equivalente ao DBML |
| `trips` | ❌ Não existe | Módulo stub; tabela não criada |
| `user_trips` | ❌ Não existe | — |
| `group_trips` | ❌ Não existe | — |
| `location_events` | ✅ Implementada (nova) | Não estava no DBML — substitui os campos `location_data` inline |
| `location_event_groups` | ✅ Implementada (nova) | Vincula eventos a grupos |
| `location_event_trips` | ✅ Criada (nova, stub) | FK para `trips` pendente |
