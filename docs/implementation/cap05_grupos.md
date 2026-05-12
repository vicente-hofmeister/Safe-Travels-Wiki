# Capítulo 5 — Módulo de Grupos

## 5.1 Visão geral

O módulo de grupos permite que usuários criem grupos de viagem, gerenciem seus membros e consultem informações do grupo. Um grupo é a unidade central que organiza o compartilhamento de localização — eventos de localização de um usuário são automaticamente vinculados a todos os grupos dos quais ele faz parte.

### Endpoints implementados

| Método | Rota                              | Descrição                                      |
| ------ | --------------------------------- | ---------------------------------------------- |
| POST   | `/group`                          | Cria novo grupo                                |
| GET    | `/group/:groupId`                 | Retorna dados e membros do grupo               |
| POST   | `/group/:groupId/members`         | Adiciona membro (apenas owner)                 |
| DELETE | `/group/:groupId/members/:userId` | Remove membro                                  |
| GET    | `/group/user/:userId`             | Lista grupos de um usuário                     |
| DELETE | `/group/:groupId`                 | Soft delete do grupo (apenas owner)            |
| GET    | `/group/:groupId/location`        | Localização mais recente dos membros (anônima) |

O endpoint `GET /group/:groupId/location` é documentado no Capítulo 6, pois sua lógica está implementada no serviço de localização.

---

## 5.2 Criação de grupo (`POST /group`)

### Body da requisição

```json
{
  "name": "Viagem PUCRS 2026",
  "description": "opcional"
}
```

| Campo         | Tipo   | Obrigatório | Validação                 |
| ------------- | ------ | ----------- | ------------------------- |
| `name`        | string | Sim         | min 1, max 150 caracteres |
| `description` | string | Não         | max 500 caracteres        |

### Resposta (201)

```json
{
  "status": "ok",
  "data": {
    "groupId": "uuid",
    "name": "Viagem PUCRS 2026",
    "description": null,
    "owner": {
      "userId": "uuid",
      "username": "alice",
      "name": "Alice Silva"
    },
    "createdAt": "2026-04-19T10:00:00Z",
    "updatedAt": "2026-04-19T10:00:00Z"
  }
}
```

---

## 5.3 Consulta de grupo (`GET /group/:groupId`)

Retorna os dados do grupo e a lista completa de membros, ordenada por data de entrada.

```json
{
  "status": "ok",
  "data": {
    "groupId": "uuid",
    "name": "Viagem PUCRS 2026",
    "description": null,
    "owner": { "userId": "...", "username": "alice", "name": "Alice Silva" },
    "members": [
      {
        "userId": "...",
        "username": "alice",
        "name": "Alice Silva",
        "joinedAt": "..."
      },
      {
        "userId": "...",
        "username": "bob",
        "name": "Bob Santos",
        "joinedAt": "..."
      }
    ],
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

Retorna `404` se o grupo não existir ou tiver sido deletado (`deleted_at IS NOT NULL`).

---

## 5.4 Gerenciamento de membros

### Adicionar membro (`POST /group/:groupId/members`)

Apenas o owner do grupo pode adicionar membros. A tentativa por qualquer outro usuário retorna `403`.

```json
{ "userId": "uuid-do-usuario-a-adicionar" }
```

A inserção usa `ON CONFLICT DO NOTHING` — adicionar um usuário que já é membro é silenciosamente ignorado.

### Remover membro (`DELETE /group/:groupId/members/:userId`)

As regras de permissão são:

| Solicitante     | Alvo                     | Resultado                                                 |
| --------------- | ------------------------ | --------------------------------------------------------- |
| Owner           | Outro membro             | Permitido                                                 |
| Qualquer membro | Si mesmo (sair do grupo) | Permitido                                                 |
| Owner           | Si mesmo                 | **Bloqueado** — owner não pode sair; deve deletar o grupo |
| Não membro      | Qualquer                 | `403 Forbidden`                                           |

---

## 5.5 Grupos de um usuário (`GET /group/user/:userId`)

Retorna todos os grupos dos quais o usuário é membro, excluindo grupos deletados, ordenados por data de entrada (mais recente primeiro).

```json
{
  "status": "ok",
  "data": [
    {
      "groupId": "uuid",
      "name": "Viagem PUCRS 2026",
      "owner": { ... },
      "joinedAt": "2026-04-19T10:00:00Z",
      "createdAt": "...",
      "updatedAt": "..."
    }
  ]
}
```

---

## 5.6 Deleção de grupo (`DELETE /group/:groupId`)

Apenas o owner pode deletar o grupo. A operação é um **soft delete**:

```sql
UPDATE groups SET deleted_at = now(), updated_at = now() WHERE group_id = $1
```

O registro permanece no banco com `deleted_at` preenchido. Todas as consultas subsequentes filtram por `deleted_at IS NULL`, efetivamente ocultando o grupo. Os dados de localização vinculados ao grupo por `location_event_groups` são preservados.

---

## 5.7 Tratamento de erros

O módulo define `GroupError`, que estende `Error` com um campo `statusCode`. O controller captura instâncias de `GroupError` e retorna o código HTTP correspondente. Os códigos utilizados são:

| Situação             | HTTP |
| -------------------- | ---- |
| Input inválido       | 400  |
| Grupo não encontrado | 404  |
| Sem permissão        | 403  |
| Erro interno         | 500  |
