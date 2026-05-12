# Capítulo 7 — Aplicativo Mobile

## 7.1 Visão geral

O aplicativo mobile do Safe Travels é desenvolvido em **React Native 0.81.5** com **Expo SDK 54**, utilizando TypeScript como linguagem. A escolha do Expo como plataforma de desenvolvimento simplifica o ciclo de build e distribuição, especialmente pela integração com o **EAS Build** (Expo Application Services) que permite gerar APKs sem necessidade de ambiente Android Studio na máquina do desenvolvedor.

A nova arquitetura do React Native (`newArchEnabled: true`) está habilitada no `app.json`, utilizando o Fabric renderer e o JSI (JavaScript Interface) para comunicação mais eficiente entre JavaScript e código nativo.

---

## 7.2 Estrutura do projeto

```
App.tsx                                  # Entry point — carrega task de background antes do React
src/
  app/
    navigation/
      RootNavigator.tsx                  # Stack: autenticação → tabs
      TabNavigator.tsx                   # Bottom tabs: Home, Trips, Map, Groups, Profile
      navigationRef.ts                   # Ref para navegação fora de componentes
    screens/
      LoginScreen.tsx                    # Tela inicial (logo + botões entrar/cadastrar)
      LoginFormScreen.tsx                # Formulário de login
      RegisterScreen.tsx                 # Formulário de cadastro
      HomeScreen.tsx                     # Tela principal (coordenadas capturadas)
      MapScreen.tsx                      # Google Maps com pins de usuários e grupos
      TripsScreen.tsx                    # Stub
      GroupsScreen.tsx                   # Stub
      ProfileScreen.tsx                  # Stub
    components/
      PasswordInput.tsx                  # Campo de senha com toggle de visibilidade
    services/
      api/
        apiFetch.ts                      # fetch com Authorization header automático
      auth/
        authApi.ts                       # HTTP: login e register (timeout 30s)
        authService.ts                   # Orquestra login/register + storage
        authStorage.ts                   # AsyncStorage: token + user
        index.ts
      location/
        locationTrackingService.ts       # Singleton: foreground e background tracking
        backgroundLocationTask.ts        # defineTask para expo-task-manager
        locationApi.ts                   # POST /location/register, GET /location/latest
        locationStorage.ts               # AsyncStorage: histórico de pontos
        locationTypes.ts                 # Tipos: StoredLocationPoint, etc.
        index.ts
      group/
        groupApi.ts                      # GET /group/user/:id, GET /group/:id/location
  theme/                                 # Design tokens: colors, spacing, typography
assets/
  fonts/                                 # Montserrat (regular e itálico)
  images/                                # Ícones e logo
scripts/
  update-env.js                          # Detecta IPv4 local e atualiza .env
  setup-firewall.ps1                     # Abre portas 3000 e 8081 no firewall Windows
  patch-expo-cache.js                    # Corrige bug de body duplo no cache do Expo CLI
```

---

## 7.3 Navegação

O aplicativo usa duas camadas de navegação com React Navigation 7:

### `RootNavigator` (Stack)

Gerencia o fluxo de autenticação. Ao inicializar, verifica se há um token armazenado no AsyncStorage:

- Token presente → navega diretamente para `Home` (TabNavigator)
- Sem token → abre a tela `Login`

```
Login
  └── LoginForm  (formulário de credenciais)
  └── Register   (cadastro)
Home → TabNavigator
```

### `TabNavigator` (Bottom Tabs)

Cinco abas na barra inferior:

| Aba | Tela | Status |
|---|---|---|
| Home | `HomeScreen` | Implementada |
| Trips | `TripsScreen` | Stub |
| Map | `MapScreen` | Implementada |
| Groups | `GroupsScreen` | Stub |
| Profile | `ProfileScreen` | Stub |

---

## 7.4 Autenticação no mobile

### `authApi.ts`

Implementa as chamadas HTTP de login e register com timeout de 30 segundos via `AbortController`. Retorna mensagens de erro legíveis ao usuário (cold start do Render pode levar até 1 minuto — o timeout longo é intencional).

### `authStorage.ts`

Armazena no AsyncStorage:
- `accessToken` — JWT para as requisições
- `storedUser` — objeto `{ id, name, username, email }` para uso em outras partes do app

### `apiFetch.ts`

Wrapper sobre `fetch` que injeta automaticamente o header `Authorization: Bearer <token>` lido do AsyncStorage, evitando repetição em cada chamada de API.

---

## 7.5 Design e tema

O sistema de design é centralizado em `src/theme/`:

- `colors.ts` — paleta de cores nomeadas (`primary`, `secondary_3`, `neutral_7`, `auxiliary_1`, etc.)
- `spacing.ts` — escala de espaçamentos
- `typography.ts` — estilos de texto (base no Montserrat)
- `base.ts` — configurações globais (background color)

A fonte padrão é **Montserrat** (variável, regular e itálico), carregada no `App.tsx` via `useFonts`.

Estilos são declarados com `StyleSheet.create` — sem styled-components ou bibliotecas de UI externas.

---

## 7.6 CI/CD — GitHub Actions + EAS Build

Dois workflows automatizados em `.github/workflows/`:

### `build-development.yml`

- **Trigger:** Pull Request aberto para `main`
- **O que faz:** Gera APK de dev client via EAS Build
- **API utilizada:** Local (via Metro do `npm start`) — útil para testar com background location

### `build-preview.yml`

- **Trigger:** Push para `main`
- **O que faz:** Gera APK de preview + cria GitHub Release automaticamente
- **API utilizada:** Render (produção) — URL configurada diretamente no `eas.json`

O `EXPO_TOKEN` (token de acesso à conta expo.dev) é armazenado como secret no repositório GitHub para autenticar o EAS CLI nos workflows.

---

## 7.7 Variáveis de ambiente

| Variável | Descrição | Configuração |
|---|---|---|
| `EXPO_PUBLIC_API_URL` | URL base da API | `.env` (local) ou `eas.json` (builds) |

`npm start` executa `scripts/update-env.js` antes de iniciar o Metro, detectando automaticamente o IPv4 local e atualizando o `.env`. Isso elimina a necessidade de configuração manual para desenvolvimento no mesmo dispositivo/rede.

Para apontar para a API em produção: `npm run start:remote` sobrescreve o `.env` com a URL do Render.

---

## 7.8 Telas implementadas

### `LoginScreen`

Tela inicial com logo e dois botões: "Entrar" (navega para `LoginForm`) e "Cadastrar" (navega para `Register`).

### `LoginFormScreen`

Formulário com campos de email e senha. Chama `authService.login()` e navega para `Home` em caso de sucesso.

### `RegisterScreen`

Formulário com campos de nome, username, email e senha. Chama `authService.register()` e navega para `Home` em caso de sucesso.

### `HomeScreen`

Exibe as coordenadas GPS capturadas mais recentes. Ponto de entrada do app após login.

### `MapScreen`

Tela central do sistema — documentada em detalhe no Capítulo 9.
