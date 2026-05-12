# Capítulo 3 — Banco de Dados

## 3.1 Tecnologia e ambientes

O banco de dados utilizado é o **PostgreSQL 17**.

| Ambiente              | Instância                                 |
| --------------------- | ----------------------------------------- |
| Desenvolvimento local | `postgres:17-alpine` em Docker Compose    |
| Produção              | Neon PostgreSQL (serverless, `us-east-1`) |

A extensão `pgcrypto` é habilitada no schema para permitir a geração de UUIDs via `gen_random_uuid()`, utilizada como valor padrão da chave primária de usuários e grupos.

---

## 3.2 Schema

### `users`

Armazena os dados de cada usuário registrado.

```sql
CREATE TABLE IF NOT EXISTS users (
  user_id       varchar(120) PRIMARY KEY DEFAULT gen_random_uuid()::text,
  username      varchar(80)  NOT NULL UNIQUE,
  name          varchar(150) NOT NULL,
  email         varchar(255) NOT NULL UNIQUE,
  password_hash text         NOT NULL,
  created_at    timestamptz  NOT NULL DEFAULT now(),
  updated_at    timestamptz  NOT NULL DEFAULT now(),
  deleted_at    timestamptz
);
```

A senha nunca é armazenada em texto claro — apenas o hash gerado pelo `bcrypt` (10 rounds por padrão).

---

### `location_events`

Cada registro de posição GPS de um usuário gera uma linha nessa tabela.

```sql
CREATE TABLE IF NOT EXISTS location_events (
  location_event_id bigserial    PRIMARY KEY,
  user_id           varchar(120) NOT NULL,
  latitude          numeric(10,7) NOT NULL,
  longitude         numeric(10,7) NOT NULL,
  accuracy_meters   integer,
  captured_at       timestamptz  NOT NULL DEFAULT now(),
  created_at        timestamptz  NOT NULL DEFAULT now(),
  CONSTRAINT location_events_latitude_range  CHECK (latitude  >= -90  AND latitude  <= 90),
  CONSTRAINT location_events_longitude_range CHECK (longitude >= -180 AND longitude <= 180),
  CONSTRAINT location_events_accuracy_non_negative CHECK (accuracy_meters IS NULL OR accuracy_meters >= 0)
);

CREATE INDEX IF NOT EXISTS idx_location_events_user_id_captured_at
  ON location_events (user_id, captured_at DESC);
```

`captured_at` representa o momento em que o GPS capturou a posição no dispositivo, enquanto `created_at` é o momento em que o registro chegou ao banco. O campo `accuracy_meters` é opcional — nem todos os dispositivos fornecem essa informação.

O índice composto em `(user_id, captured_at DESC)` otimiza a consulta mais frequente: buscar o último evento de cada usuário.

---

### `groups`

Grupos de viagem criados pelos usuários.

```sql
CREATE TABLE IF NOT EXISTS groups (
  group_id    varchar(120)  PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name        varchar(150)  NOT NULL,
  description varchar(500),
  owner_id    varchar(120)  NOT NULL,
  created_at  timestamptz   NOT NULL DEFAULT now(),
  updated_at  timestamptz   NOT NULL DEFAULT now(),
  deleted_at  timestamptz
);
```

Grupos utilizam **soft delete**: ao invés de remover o registro, o campo `deleted_at` é preenchido. Todas as consultas filtram por `deleted_at IS NULL`. Isso preserva o histórico de localização associado ao grupo.

---

### `group_members`

Relacionamento muitos-para-muitos entre usuários e grupos.

```sql
CREATE TABLE IF NOT EXISTS group_members (
  group_id   varchar(120) NOT NULL,
  user_id    varchar(120) NOT NULL,
  joined_at  timestamptz  NOT NULL DEFAULT now(),
  PRIMARY KEY (group_id, user_id)
);

CREATE INDEX IF NOT EXISTS idx_group_members_user_id ON group_members (user_id);
```

O índice em `user_id` otimiza a consulta "grupos do usuário" e a vinculação automática de eventos de localização.

---

### `location_event_groups`

Vincula um evento de localização a todos os grupos dos quais o usuário faz parte no momento do registro.

```sql
CREATE TABLE IF NOT EXISTS location_event_groups (
  location_event_id bigint       NOT NULL,
  group_id          varchar(120) NOT NULL,
  created_at        timestamptz  NOT NULL DEFAULT now(),
  PRIMARY KEY (location_event_id, group_id),
  CONSTRAINT fk_leg_event FOREIGN KEY (location_event_id)
    REFERENCES location_events(location_event_id) ON DELETE CASCADE,
  CONSTRAINT fk_leg_group FOREIGN KEY (group_id)
    REFERENCES groups(group_id)
);

CREATE INDEX IF NOT EXISTS idx_location_event_groups_group_id
  ON location_event_groups (group_id, location_event_id DESC);
```

O `ON DELETE CASCADE` garante que ao remover um evento de localização, os vínculos com grupos são automaticamente removidos. O índice em `(group_id, location_event_id DESC)` otimiza a consulta de localização por grupo.

---

### `location_event_trips`

Estrutura preparada para futura associação de eventos de localização a viagens. A FK para a tabela `trips` (ainda não implementada) será adicionada quando o módulo for desenvolvido.

```sql
CREATE TABLE IF NOT EXISTS location_event_trips (
  location_event_id bigint       NOT NULL,
  trip_id           varchar(120) NOT NULL,
  created_at        timestamptz  NOT NULL DEFAULT now(),
  PRIMARY KEY (location_event_id, trip_id),
  CONSTRAINT fk_let_event FOREIGN KEY (location_event_id)
    REFERENCES location_events(location_event_id) ON DELETE CASCADE
);
```

---

## 3.3 Decisões de modelagem

### `location_events` como tabela separada

O DBML original do TCC I previa um campo `location_data` inline nas tabelas `users`, `groups` e `trips`. Essa abordagem foi substituída por uma tabela dedicada `location_events` com histórico completo de posições.

A mudança decorre diretamente da remoção do Kinesis: sem o intermediário assíncrono, todo evento de localização vai diretamente ao banco. Uma tabela dedicada permite:

- Manter histórico completo de trajetórias
- Indexar eficientemente por usuário e tempo
- Vincular o mesmo evento a múltiplos grupos via tabela de junção

### `bigserial` para `location_event_id`

Eventos de localização são registrados com alta frequência (a cada 15 minutos em background, e adicionalmente ao abrir o mapa). O tipo `bigserial` (inteiro de 64 bits com auto-incremento) foi escolhido para suportar o volume esperado sem colisões e com inserção eficiente, ao contrário de UUIDs que têm custo maior de geração e índice.

### UUIDs como strings

Os IDs de `users` e `groups` são armazenados como `varchar(120)` contendo UUIDs em formato texto (gerados por `gen_random_uuid()::text`). Essa escolha evita dependência do tipo `uuid` nativo do PostgreSQL na camada da aplicação, simplificando a serialização JSON.

### Soft delete em `groups`

A remoção de um grupo não apaga o registro. Em vez disso, preenche `deleted_at`. Isso preserva o histórico de `location_event_groups` vinculado ao grupo, o que é relevante para análise de trajetórias futuras e para a tabela `location_event_trips` quando o módulo de viagens for implementado.

### Sem Foreign Keys entre `location_events` e `users`

A tabela `location_events` não define uma FK explícita para `users.user_id`. Isso permite que eventos de localização persistam mesmo se o usuário for futuramente marcado como deletado (`deleted_at`), mantendo a integridade do histórico sem complexidade adicional de regras de cascata.
