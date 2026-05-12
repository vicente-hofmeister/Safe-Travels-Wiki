# Capítulo 8 — Serviço de Localização (Mobile)

## 8.1 Visão geral

O rastreamento de localização no mobile é gerenciado pelo `LocationTrackingService`, um singleton exportado como `locationTrackingService`. Ele encapsula toda a lógica de acesso ao GPS, armazenamento local de pontos e integração com a API, expondo uma interface clara para os componentes e tarefas de background.

O serviço suporta dois modos de operação:

- **Foreground:** app em primeiro plano, usando `watchPositionAsync`
- **Background:** app minimizado, usando `expo-task-manager`

---

## 8.2 Permissões

O Android e iOS distinguem permissões de localização em foreground e background. O serviço gerencia as duas:

- `requestForegroundPermissionsAsync()` — solicitada ao abrir o mapa ou capturar posição manualmente
- `requestBackgroundPermissionsAsync()` — solicitada ao iniciar o rastreamento em background

As permissões são verificadas antes de cada operação. Se negadas, o método retorna `null` ou lança exceção com mensagem descritiva.

---

## 8.3 Foreground tracking

O modo foreground é implementado via `Location.watchPositionAsync()` com as seguintes configurações padrão:

| Parâmetro | Valor | Significado |
|---|---|---|
| `accuracy` | `Balanced` | Equilíbrio entre precisão e consumo de bateria |
| `distanceInterval` | 10 m | Atualiza ao mover pelo menos 10 metros |
| `timeInterval` | 10.000 ms | Atualiza no máximo a cada 10 segundos |
| `maxStoredPoints` | 500 | Limite de pontos no AsyncStorage |

Cada atualização de posição chama `appendStoredLocation()`, que salva o ponto no AsyncStorage (com limite de 500 entradas).

O método `startTracking()` é idempotente: se o serviço já estiver rastreando, retorna imediatamente. `stopTracking()` cancela a subscription e libera recursos.

---

## 8.4 Captura pontual de posição

O método `captureCurrentPosition()` captura a posição atual do dispositivo sem iniciar rastreamento contínuo:

```
1. Verifica permissão foreground
2. Verifica se serviços de localização estão ativos no dispositivo
3. Tenta getCurrentPositionAsync(Balanced)
4. Se falhar (timeout, ausência de sinal), tenta getLastKnownPositionAsync
   com maxAge de 5 minutos e requiredAccuracy de 500 metros
5. Salva o ponto no AsyncStorage (erros de storage são ignorados)
6. Retorna o ponto
```

O método `registerCurrentPosition(userId)` combina captura e envio para a API em uma única chamada.

---

## 8.5 Background tracking

O rastreamento em background utiliza `expo-task-manager`, que permite executar código JavaScript mesmo com o app minimizado, desde que seja um build nativo (APK gerado por EAS Build). O Expo Go não suporta esse recurso.

### Configuração

```
accuracy:         Balanced
distanceInterval: 50 m
timeInterval:     15 minutos (900.000 ms)
```

Os parâmetros de background são mais conservadores que o foreground para minimizar consumo de bateria. O intervalo de 15 minutos é suficiente para o contexto de uma viagem em grupo — localização com granularidade de minutos é adequada para o caso de uso.

No Android, o sistema exige um **foreground service** para execução em background. Uma notificação persistente é exibida ao usuário:

> **Safe Travels** — "Compartilhando localização com o grupo."

com cor `#4E3FCA` (primary da paleta do app).

### Inicialização

```
startBackgroundTracking():
  1. Solicita requestBackgroundPermissionsAsync()
  2. Verifica se a task já está rodando (hasStartedLocationUpdatesAsync)
  3. Se não, chama startLocationUpdatesAsync(BACKGROUND_LOCATION_TASK, config)
```

O método é idempotente: se a task já estiver ativa, retorna sem fazer nada.

---

## 8.6 `backgroundLocationTask`

A task de background é definida em `src/app/services/location/backgroundLocationTask.ts` e registrada com `TaskManager.defineTask()`:

```
Ao receber atualização de localização:
  1. Verifica se há erro — loga e retorna se sim
  2. Extrai o array locations da data
  3. Recupera o usuário logado do AsyncStorage (getStoredUser)
  4. Se não há usuário, retorna sem fazer nada
  5. Pega a última posição do batch (locations[length - 1])
  6. Chama registerLocation() para enviar à API
  7. Erros de rede são capturados e logados sem travar a task
```

**Requisito crítico:** esse arquivo deve ser importado no topo de `App.tsx` **antes de qualquer render do React**, conforme exigência do `expo-task-manager`. Importá-lo após a inicialização do React resulta em erro de "task não registrada".

```typescript
// App.tsx — primeira linha
import "./src/app/services/location/backgroundLocationTask";
```

---

## 8.7 Armazenamento local (`locationStorage.ts`)

Pontos de localização são persistidos no AsyncStorage sob a chave `stored_locations`. As operações disponíveis:

| Função | Descrição |
|---|---|
| `appendStoredLocation(point, max)` | Adiciona ponto; remove os mais antigos se ultrapassar o limite |
| `readStoredLocations()` | Retorna todos os pontos armazenados |
| `clearStoredLocations()` | Remove todos os pontos |

O limite de 500 pontos evita crescimento ilimitado do AsyncStorage. Com o intervalo de background de 15 minutos, 500 pontos representam aproximadamente 5 dias de rastreamento contínuo.

---

## 8.8 Interface pública do serviço

| Método | Descrição |
|---|---|
| `requestPermissions()` | Solicita permissão foreground; retorna boolean |
| `captureCurrentPosition()` | Captura posição atual; salva localmente |
| `registerCurrentPosition(userId)` | Captura + envia para a API |
| `startTracking(options?)` | Inicia foreground tracking |
| `stopTracking()` | Para foreground tracking |
| `startBackgroundTracking()` | Inicia background tracking via task manager |
| `stopBackgroundTracking()` | Para background tracking |
| `isBackgroundTrackingActive()` | Verifica se background tracking está ativo |
| `getStoredLocations()` | Retorna pontos do AsyncStorage |
| `clearStoredLocations()` | Limpa storage local |
