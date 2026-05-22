# Feature Conexões — Especificação V2

> Documento de referência. Atualizar conforme decisões.

## 1. 🎯 Visão geral

Permitir que um usuário compartilhe **alertas de presença** com pessoas de
confiança (família, amigos), sem expor sua localização contínua. Quando o
usuário entra ou sai de um raio configurado **e a conexão consentiu em receber
notificações desse raio**, ela recebe um push ("Maria chegou no Trabalho").

**Princípios:**
- 🔒 **Duplo consentimento**: A escolhe enviar para B, e B precisa aceitar receber.
- 🔇 **Anti-flood**: máximo de 2 raios **por lugar** notificando uma mesma conexão.
- 🔒 Privacidade: nenhuma localização contínua é compartilhada — só o evento.
- ⚡ Real-time via FCM (Firebase Cloud Messaging).
- 💸 Recurso PRO (driver de monetização).
- 🔁 Conexão em si: bidirecional, ambos aceitam.

## 2. 👥 Modelo de dados (Firestore)

### `users/{uid}`
```ts
{
  email: string,           // lowercase, indexado (queryable)
  displayName: string,
  photoURL?: string,
  fcmTokens: string[],     // multi-device
  // ...campos atuais (places, expireAt, isPro, etc.)
}
```

### `connections/{connectionId}`
ID = `${uidA}_${uidB}` (ordenado alfabeticamente, idempotente).
```ts
{
  participants: [uidA, uidB],
  participantsInfo: {
    [uidA]: { displayName, photoURL, email },
    [uidB]: { displayName, photoURL, email }
  },
  status: 'pending' | 'accepted' | 'blocked',
  initiator: uid,
  createdAt: timestamp,
  acceptedAt?: timestamp,
  expireAt: timestamp      // TTL: 5 meses (renovado a cada interação se accepted)
}
```

### `notificationSubscriptions/{subscriptionId}` ⭐
Modela o **consentimento de B em receber alertas dos raios de A**.
ID = `${ownerUid}_${subscriberUid}_${placeId}_${ringId}`.
```ts
{
  ownerUid: string,        // dono do lugar (A — gera o trigger)
  subscriberUid: string,   // recebe push (B)
  placeId: string,
  ringId: string,
  placeName: string,       // snapshot
  ringDistance: number,
  status: 'pending' | 'accepted' | 'rejected',
  createdAt: timestamp,
  respondedAt?: timestamp,
  expireAt: timestamp      // TTL: 5 meses (renovado a cada notificação enviada)
}
```

### `notifications/{notifId}` (histórico — entra no V2)
```ts
{
  toUid: string,
  fromUid: string,
  fromName: string,
  placeName: string,
  event: 'enter' | 'exit',
  ringDistance: number,
  createdAt: timestamp,
  read: boolean,
  expireAt: timestamp      // TTL: 5 meses
}
```

## 3. 🔐 Regras Firestore

```
users/{uid}:
  read: request.auth.uid == uid
        OR (já é conexão accepted entre os dois)
        OR (consulta por email — só campo email, com rate-limit)
  write: request.auth.uid == uid

connections/{connId}:
  read: request.auth.uid in resource.data.participants
  create: request.auth.uid == request.resource.data.initiator
          AND request.auth.uid in request.resource.data.participants
          AND status == 'pending'
  update: request.auth.uid in resource.data.participants
          AND mudança limitada a {status, acceptedAt}
  delete: request.auth.uid in resource.data.participants

notificationSubscriptions/{subId}:
  read: request.auth.uid in [resource.data.ownerUid, resource.data.subscriberUid]
  create: request.auth.uid == request.resource.data.ownerUid
          AND existe connection accepted entre ownerUid e subscriberUid
          AND status == 'pending'
          AND (count subs ativas ownerUid→subscriberUid no MESMO placeId) <= 2
  update: request.auth.uid == resource.data.subscriberUid
          AND mudança limitada a {status, respondedAt}
  delete: request.auth.uid in [resource.data.ownerUid, resource.data.subscriberUid]

notifications/{notifId}:
  read: request.auth.uid == resource.data.toUid
  create: false (só Cloud Function)
  update: request.auth.uid == resource.data.toUid (só `read`)
  delete: request.auth.uid == resource.data.toUid
```

## 4. 🔄 Fluxos do usuário

### 4.1. Convidar conexão (por e-mail)
1. Em **Configurações → Conexões → Convidar amigo**.
2. Digita o e-mail.
3. App resolve email → uid:
   - Não existe → "Esse e-mail ainda não usa o Geo Alert. Convide a baixar."
     + botão **Compartilhar** (deep link / share sheet).
   - Existe → cria `connections/{ordered}` com `status: pending`.
4. Cloud Function `onConnectionCreated` → push pra B: "Pedro quer se conectar".

### 4.2. Aceitar/Recusar conexão
1. B abre o app → HomeScreen detecta convite (listener Firestore).
2. Banner: "Pedro quer se conectar [Aceitar] [Recusar]".
3. **Aceitar** → `status: accepted`. Push pra A: "Pedro aceitou".
4. **Recusar** → delete doc.

### 4.3. Configurar alerta em um raio (lado A)
Em **ConfigureRingsScreen**, cada raio ganha:
```
🔔 Avisar conexões neste raio              [Toggle]
   ↳ [chip Maria] [chip João]  + Adicionar
```

Ao selecionar uma conexão num raio:
- **Validação local (UI)**: se já existem **2 raios neste lugar** com essa
  conexão como subscriber ativo, bloqueia:
  > "Você já tem 2 raios deste lugar avisando *Maria*. Remova um para
  > adicionar outro."
- Senão → cria `notificationSubscriptions/{...}` com `status: pending`.
- Cloud Function `onSubscriptionCreated` → push pra B com **contexto**
  (item 4.4).

### 4.4. Aceitar/recusar receber alerta (lado B)
- B recebe push → "Pedro quer te avisar quando passar pelo *Trabalho* (500m)".
- Em **Configurações → Conexões → Solicitações**, mostra:
  ```
  Pedro
  Trabalho · raio 500m
  ⚠️ Você já tem 3 alertas ativos do Pedro neste lugar
  [Aceitar]  [Recusar]
  ```
  > A linha "⚠️" é um **resumo de transparência**: conta quantos alertas B já
  > recebe daquele owner naquele place, ajudando B a decidir.
- Aceitar → `status: accepted`. A vê: ✅ Maria.
- Recusar → `status: rejected`. A vê: ⛔ Maria recusou.
- B pode revogar uma sub aceita a qualquer momento (delete doc).

### 4.5. Disparo (geofencing)
Quando A entra/sai de um raio:
```
geofencing.ts → onEnter/onExit:
  if (ring tem subs accepted):
    httpsCallable('sendGeofenceNotification')({placeId, ringId, event})
```

Cloud Function:
1. Carrega lugar/raio do `users/{ownerUid}`.
2. Busca `notificationSubscriptions` accepted desse `(ownerUid, placeId, ringId)`.
3. Para cada `subscriberUid`:
   - Busca `fcmTokens`.
   - Envia push: "*Pedro* chegou no *Trabalho*".
   - Cria `notifications/{id}` (histórico).

### 4.6. Recebimento (lado B)
- Foreground → toast in-app + atualiza badge.
- Background → notificação nativa.
- Tap → tela de histórico de notificações.

## 5. 🚦 Limites e enforcement

### Por A (envia)
- Máx **2 raios por lugar** alertando uma mesma conexão.
- 3 camadas de validação:
  1. **UI**: bloqueia o toggle/seleção com mensagem amigável.
  2. **Firestore rules**: count ≤ 2 por (owner, subscriber, place).
  3. **Cloud Function** (`onSubscriptionCreated`): valida e recusa se passar.

### Por B (recebe)
- Aceita/recusa cada solicitação individualmente.
- Pode revogar a qualquer momento.
- Mute por conexão inteira → futuro.

### Transparência ao B na tela de aceite
- Mostrar: "Você já tem N alertas ativos de A neste lugar".
- Mostrar: "Total de alertas que você recebe de A: M".

## 6. 🛠️ Arquitetura técnica

### Cliente
```
src/
├── services/
│   ├── connections.ts          NOVO   invite por email, accept, list
│   ├── subscriptions.ts        NOVO   manage notificationSubscriptions
│   ├── pushTokens.ts           NOVO   register/refresh FCM
│   └── geofencing.ts           MOD    chamar sendGeofenceNotification
├── types/index.ts              MOD    Connection, NotificationSubscription
├── screens/
│   ├── ConnectionsScreen.tsx   NOVO   convidar, lista, solicitações
│   ├── ConfigureRingsScreen    MOD    toggle + chips + enforcement
│   ├── SettingsScreen          MOD    link + badge pendências
│   └── HomeScreen              MOD    banner convites pendentes
└── i18n/locales/{pt,en}.ts     MOD    chaves connections.* / subscriptions.*
```

### Servidor
```
firestore.rules                 4 coleções (com count ≤ 2 por place)
firebase.json                   targets functions + firestore
functions/
├── package.json                deps: firebase-admin, firebase-functions
└── index.js                    funções:
    ├─ onConnectionCreated     push "fulano quer se conectar"
    ├─ onConnectionAccepted    push "fulano aceitou"
    ├─ onSubscriptionCreated   push "fulano quer te avisar do raio X"
    ├─ onSubscriptionAccepted  push "fulano aceitou ser avisado"
    └─ sendGeofenceNotification (HTTPS callable) — A entra/sai → push pra subs
```

## 7. 🔒 Privacidade
- Sem localização contínua compartilhada.
- Listas de conexões e subs só visíveis para os 2 envolvidos.
- E-mail só é exposto na busca para o iniciador (rate-limit + abuse detection).
- **TTL global de 5 meses** em todas as coleções (`connections`, `notificationSubscriptions`, `notifications`) via campo `expireAt` + Firestore TTL policy. Em registros ativos (`accepted`), `expireAt` é renovado em cada interação (notificação enviada / aceite / atualização).
- Bloquear/excluir conexão remove todas as subs entre eles (cleanup function).

## 8. 💰 Modelo de tier

| Recurso                              | Anonymous | Free | PRO |
|---|---|---|---|
| Criar conexões                       | ❌ | até 2 | ilimitado |
| Receber alertas                      | ❌ | ✅   | ✅ |
| Enviar alertas (criar subscriptions) | ❌ | ❌   | ✅ |
| Raios alertando por conexão (place)  | — | —   | máx 2 |

Free **recebe** mas não **envia** — gatilho social pra upgrade.

## 9. 📅 Roadmap

### Fase 1 — Fundação
- [ ] Tipos `Connection`, `NotificationSubscription`, `FcmToken`
- [ ] i18n keys (pt + en)
- [ ] `pushTokens.ts` (register FCM no login)
- [ ] Plugin messaging no `app.json`
- [ ] APNs cert (a partir do zero — ver §11)
- [ ] `firebase.json` + `firestore.rules` + deploy rules
- [ ] `firestore.indexes.json` (queries por email, por participants)

### Fase 2 — Convidar/aceitar conexão
- [ ] `connections.ts` (invite por email, accept, list, listen)
- [ ] `ConnectionsScreen.tsx`
- [ ] Atalho em SettingsScreen
- [ ] Banner em HomeScreen
- [ ] Cloud Functions: `onConnectionCreated`, `onConnectionAccepted`

### Fase 3 — Subscriptions de raio
- [ ] `subscriptions.ts` (CRUD + validar limite ≤ 2 por place)
- [ ] ConfigureRingsScreen: toggle + chips + enforcement
- [ ] Tela "Solicitações pendentes" (com contadores de transparência §5)
- [ ] Cloud Functions: `onSubscriptionCreated`, `onSubscriptionAccepted`

### Fase 4 — Disparo + histórico + polish
- [ ] Cloud Function `sendGeofenceNotification` (HTTPS callable)
- [ ] Wiring no `geofencing.ts`
- [ ] Histórico de notificações (tela)
- [ ] Aplicar limites de tier
- [ ] Badge de pendências
- [ ] Telemetria (count pushes/dia, custo)

## 10. ⚠️ Riscos
1. Custo Cloud Functions / FCM → rate-limit por A (ex: 100 pushes/dia).
2. iOS background → validar `headlessTask` chamando HTTPS callable.
3. APNs setup (ver §11).
4. Token FCM expirado → função remove inválidos do array.
5. Conta excluída → cleanup de subs/connections órfãos (`onUserDelete`).
6. E-mail privacy → query expõe existência de conta. Mitigar com Cloud Function dedicada + captcha de uso.

## 11. 🚀 Setup do zero (APNs + Firebase Functions)

> Estamos partindo do zero. Documentar passo a passo aqui ao executar.

### APNs (iOS push)
1. **Apple Developer** → Certificates, Identifiers & Profiles
2. Identifiers → bundle `com.silvars.geoalert` → habilitar **Push Notifications**
3. Keys → `+` → criar **APNs Auth Key** → baixar `.p8`
   - Anotar `Key ID` e `Team ID`
4. **Firebase Console** → projeto `geo-alert-6c2d4`:
   - Project settings → Cloud Messaging → Apple app config
   - Upload `.p8` + Key ID + Team ID

### Cloud Functions
1. `firebase init functions` (linguagem: JavaScript ou TypeScript — JS pra simplicidade)
2. Reaproveitar `functions/` já existente (vazio)
3. Deps: `firebase-admin`, `firebase-functions`
4. `firebase deploy --only functions:onConnectionCreated`

### Firestore Rules
1. `firebase deploy --only firestore:rules`
2. Validar com emulator antes em dev

## 12. ✅ Critérios de aceite (V2)
- [ ] A convida B por e-mail; B aceita
- [ ] A liga "Avisar Maria" num raio; Maria aceita
- [ ] Tela de aceite mostra contador de transparência
- [ ] A entra no raio → B recebe push em até 5s
- [ ] A sai → B recebe push de saída
- [ ] A tenta colocar Maria num 3º raio do mesmo lugar → bloqueado com msg
- [ ] B revoga sub → próximos triggers não enviam push
- [ ] App fechado em B → push na bandeja
- [ ] App aberto em B → toast + entrada no histórico
- [ ] Tier limites aplicados
- [ ] Bloquear conexão remove subs entre os dois
