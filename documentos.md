# Corrida+ — Documentação técnica

Este ficheiro reúne a documentação de arquitetura que antes vivia como comentários
longos dentro do `corridaplus.html`. O HTML principal mantém apenas comentários
curtos, apontando para aqui quando for preciso mais contexto.

## Índice
1. [Estado global e chaves de mês](#1-estado-global-e-chaves-de-mês)
2. [Sincronização com a nuvem](#2-sincronização-com-a-nuvem)
3. [Pull-antes-de-push (merge)](#3-pull-antes-de-push-merge)
4. [IVA, comissão e cálculo de lucro](#4-iva-comissão-e-cálculo-de-lucro)
5. [Encriptação opcional (Modo Dev)](#5-encriptação-opcional-modo-dev)
6. [Desbloqueio biométrico (WebAuthn)](#6-desbloqueio-biométrico-webauthn)
7. [Easter eggs](#7-easter-eggs)
8. [Log de sincronização](#8-log-de-sincronização)
9. [Alterações desta versão (v3.1.0)](#9-alterações-desta-versão-v310)

---

## 1. Estado global e chaves de mês

O app organiza tudo por mês usando uma string `"AAAA-MM"` (ex: `"2026-07"`) como
chave do objeto global `monthData`. Cada entrada de mês tem a forma:

```js
{ earnings: [], billOverrides: {}, billsPaid: {}, hiddenBills: {}, variableExpenses: [] }
```

- `earnings` — ganhos do dia a dia (Uber, Bolt, Particular, Outros)
- `variableExpenses` — despesas variáveis (combustível, manutenção, etc.)
- `billOverrides` — valor customizado de uma despesa fixa **só neste mês**
- `billsPaid` — registo de pagamento de despesas fixas neste mês
- `hiddenBills` — despesas fixas ocultadas neste mês (sem apagar globalmente)

`ensureMonthEntry(key)` garante que a entrada existe antes de qualquer leitura —
isto é chamado constantemente, inclusive só por navegação, sem o utilizador ter
adicionado dados. Essas entradas "vazias" (criadas só por navegar) **não** são
enviadas à nuvem — `pruneEmptyMonths()` remove-as do payload antes do POST.

`bills` (despesas fixas) é uma lista global, não por mês. Cada bill tem
`createdMonthKey` — o mês em que foi criada — para não aparecer retroativamente
em meses anteriores. `frequency` pode ser `diaria`, `semanal` ou `mensal`, e
`totalFixedBills()` multiplica o valor pelos dias/semanas do mês corrente
conforme o caso.

## 2. Sincronização com a nuvem

Arquitetura: um Google Apps Script (fora deste ficheiro) expõe:
- `doGet()` — devolve o conteúdo da célula A1 como JSON
- `doPost()` — sobrescreve essa célula com o body recebido

O app só faz GET e POST para a URL guardada em `syncUrl` (device-local, nunca
enviada como parte do payload). Não há autenticação própria — a "segurança"
vem de a URL do Apps Script ser secreta.

- **Timeout**: toda chamada de rede tem limite de 15s (`SYNC_TIMEOUT_MS`) via
  `AbortController`.
- **Retry automático**: falhas muito rápidas (<500ms) são tratadas como
  sintoma de rate-limiting/cold-start do Apps Script e disparam até 2 retries
  com backoff crescente (800ms, depois 1600ms) antes de desistir.
- **Migração de planilha**: se o JSON recebido tiver um campo `syncUrl` em
  texto plano (verificado *antes* de tentar decifrar, mesmo com encriptação
  ativa), o app troca de planilha e sincroniza de novo com a URL nova.
- **Encriptação**: se o JSON vier com `{encrypted:true, salt, iv, data}`, ver
  secção 5.
- **Log**: cada tentativa é registada via `logSyncEvent()` — ver secção 8.

`pushToCloud()` sobrescreve o documento inteiro (não é merge incremental do
lado do servidor) — por isso a secção seguinte é importante.

## 3. Pull-antes-de-push (merge)

**Comportamento a partir da v3.1.0:** antes de qualquer envio à nuvem, o app
primeiro busca (GET) o estado mais recente da planilha e faz merge com o
estado local, e só então monta e envia o payload (POST).

Isto evita o cenário em que dois dispositivos editam quase ao mesmo tempo e
um deles sobrescreve silenciosamente o trabalho do outro (last-write-wins
"cego"). Fluxo em `pushToCloud()`:

1. `pullLatestAndMergeBeforePush()` — GET com timeout curto (6s,
   `PRE_PUSH_PULL_TIMEOUT_MS`), separado do timeout principal de push.
2. Se a resposta vier encriptada ou com uma migração de URL pendente, o pull
   é ignorado silenciosamente (não faz sentido interromper o utilizador com
   um pedido de senha só para mandar uma edição em segundo plano) — o push
   segue com os dados locais.
3. Caso contrário, `mergeRemoteIntoLocal(data)` aplica o remoto por cima do
   local:
   - **bills**: união por `id` — bills que existem só na nuvem (criadas
     noutro dispositivo) são adicionadas; bills que existem só localmente
     são mantidas (o utilizador pode estar a criar uma agora mesmo).
   - **earnings / variableExpenses** (por mês): união por `id` — cada
     registo tem um `uid()` único, então não há colisão real entre
     dispositivos; a lista final é a soma dos dois lados.
   - **billOverrides / billsPaid / hiddenBills**: merge de objeto,
     `{...remoto, ...local}` — o valor local prevalece em caso de conflito
     na mesma chave (edição feita agora mesmo, neste dispositivo).
   - **profile**: `{...remoto, ...local}` — campos que o dispositivo local
     não tem são preenchidos pelo remoto; campos que o local já tem
     prevalecem.
4. Só depois disto o payload final (`{bills, monthData, profile}`) é
   construído e enviado via POST.
5. Se o pull falhar (rede lenta, offline), o push segue com os dados locais
   como antes — não bloqueia a gravação do utilizador.

Isto não é um sistema de merge com controlo de versão de verdade (não há
vetor de relógio nem resolução de conflito campo-a-campo em profile), mas
reduz bastante o risco de perda de dados em edições quase-simultâneas,
porque a maior parte do estado (earnings, expenses, bills) é merge por união
de IDs, não substituição.

### 3.1 Lápides de exclusão ("tombstones")

União por ID tem um efeito colateral perigoso: apagar um ganho localmente
e depois correr o merge de pré-push faz esse ganho **voltar**, porque ele
ainda existe na planilha remota e o merge não tem como saber que "existe
remotamente, não existe localmente" significa "apagado aqui" em vez de
"criado noutro dispositivo".

A correção é um conjunto `deletedIds` (persistido em `window.storage`,
device-local, nunca sincronizado):

- Cada função de exclusão (`removeEarning`, `deleteVariableExpense`,
  `deleteBill`, `unpayBill`, `restoreHiddenBill`) chama `markDeleted(key)`
  com uma chave identificando o que foi apagado — `earning:<id>`,
  `expense:<id>`, `bill:<id>`, `billsPaidKey:<mês>:<billId>`,
  `hiddenBillsKey:<mês>:<billId>`.
- `mergeRemoteIntoLocal()` ignora qualquer item remoto cuja chave esteja em
  `deletedIds` — ou seja, não volta a inserir algo que foi apagado aqui,
  mesmo que ainda exista na nuvem.
- As lápides são limpas automaticamente assim que um push tiver sucesso
  (`clearDeletedIdsAfterSuccessfulPush()`) — nesse ponto a nuvem já reflete
  a exclusão, então não há mais risco do item "voltar", e a lista não
  cresce indefinidamente.
- **"Apagar todos os dados"** (`confirmResetAll()`) é tratado como caso
  especial: em vez de um push normal (que faria merge e traria tudo de
  volta da nuvem antes de enviar o estado vazio), chama
  `pushToCloud(undefined, true)` — o segundo parâmetro (`skipMerge`) pula o
  merge de pré-push inteiramente, porque apagar tudo é uma sobrescrita
  intencional, não uma edição incremental.

## 4. IVA, comissão e cálculo de lucro

A fórmula usada em todas as telas (Ganhos, Despesas, Estatísticas, Lucro):

```
líquido = (bruto Uber+Bolt − IVA − comissão) − combustível + Particular/Outros
lucro   = líquido − despesas fixas − outras despesas variáveis
```

- IVA e comissão incidem **só** sobre o bruto de Uber+Bolt, nunca sobre
  Particular/Outros — essa separação é feita por quem chama
  `ivaAmount()`/`comissaoAmount()`, não dentro dessas funções.
- Ambas as taxas vivem em `profile.ivaRate`/`profile.comissaoRate`
  (percentagem inteira, ex: `6` = 6%), sincronizadas como parte do profile.
- Se mudares esta fórmula, confirma consistência entre `renderMonthSummary()`,
  `renderLucroMonth()`/`lucroBreakdownHTML()` e `renderStatsMonth()` — as três
  implementam o mesmo cálculo separadamente.

## 5. Encriptação opcional (Modo Dev)

AES-GCM 256 bits via Web Crypto API nativa do browser, sem dependências
externas.

- A senha nunca é enviada; só a chave derivada localmente (PBKDF2, 100.000
  iterações, SHA-256) é usada para cifrar/decifrar.
- O `salt` é diferente a cada encriptação — a mesma senha nunca produz a
  mesma chave duas vezes.
- Sem a senha certa, o conteúdo da célula A1 da planilha é ilegível (só
  texto cifrado em base64).
- Ao ativar, se ainda não houver senha guardada, o app pede uma via
  `prompt()`.
- **Comportamento de sync**: se `requireUnlockEachSync` estiver desligado
  (padrão) e já houver senha guardada, o app decifra automaticamente em
  segundo plano. Se ligado, sempre pede confirmação (senha ou biometria).

## 6. Desbloqueio biométrico (WebAuthn)

A biometria nunca sai do dispositivo — o app só recebe um sim/não do
sistema operativo. A senha real de encriptação continua a ser a chave; a
biometria só decide se essa senha, já guardada localmente, pode ser
liberada nesta sessão sem digitar de novo.

- Exige encriptação já ativa com senha definida.
- Ao ativar, regista uma credencial WebAuthn "platform" (Face ID/Touch
  ID/digital) **neste dispositivo especificamente** — a credencial não é
  sincronizada, cada aparelho tem a sua própria.

## 7. Easter eggs

Dois easter eggs vivem no Modo Dev, ambos **desativados por padrão** a
partir da v3.1.0 e configurados via `profile` (sincronizado).

### 7.1 Brincadeira "hackeamento" (`runHackScreen`)

- Só roda se **ambas** as condições forem verdadeiras:
  1. `profile.easterEggDisabled === false` (ativação explícita, feita no
     interruptor do Modo Dev — o padrão de um perfil novo é `undefined`,
     que conta como desativado).
  2. O nome no perfil bate exatamente (case-insensitive) com
     `EASTER_EGG_TARGET_NAME`.
- Com as duas condições satisfeitas, ainda entra um sorteio:
  `profile.easterEggRate` (0 a 1) é a probabilidade por abertura do app
  (padrão 3% quando ativada sem taxa configurada).
- É 100% cosmético — uma animação de texto em `<div id="hackScreen">`, não
  toca em nenhum dado real.
- O texto está em Base64 só para não aparecer em leitura casual do código
  (não é segurança de verdade — qualquer pessoa pode rodar `atob()` na
  consola). Ver comentário junto a `runHackScreen()` no HTML para o
  processo de editar esse texto.

### 7.2 Mensagem de aniversário (`runBirthdayScreen`)

- Só dispara se `profile.birthdayDate` tiver sido **explicitamente
  configurada** (formato `DD-MM`) nas definições do Modo Dev. Um perfil
  novo não tem essa data definida, então a mensagem nunca aparece
  automaticamente até alguém a configurar.
- Funciona para qualquer nome preenchido no perfil (não está fixa em
  nenhum nome específico) — a tela usa `profile.name` dinamicamente.
- Tem prioridade sobre o hackeamento se caírem no mesmo dia.
- Limpar o campo de data nas definições (`setBirthdayDate('')`) desativa a
  mensagem de novo, apagando `profile.birthdayDate`.

### 7.3 Sincronização

Todas as configurações (taxa, ativado/desativado, data de aniversário)
vivem dentro de `profile`, já sincronizado inteiro. Isto é intencional: se
a pessoa descobrir e desativar num dispositivo, fica desativada em todos os
que sincronizam com a mesma planilha.

## 8. Log de sincronização

Guarda os últimos 50 eventos de sincronização (sucesso, erro, timeout,
info) localmente, útil para diagnosticar problemas sem acesso direto ao
dispositivo do utilizador. Exportável via `navigator.share()` (folha nativa
no iOS/Android) com fallback para download de `.txt`.

## 9. Atualização automática e silenciosa (na abertura)

A partir da v3.2.1, o app **não pergunta nada** — se encontrar uma versão
mais nova publicada, atualiza-se sozinho, e só avisa depois de já ter
acontecido.

Como o app é um ficheiro estático (sem service worker/PWA de verdade), a
"atualização" é simplesmente recarregar a página com bypass de cache. Isso
exige dois passos, feitos em duas aberturas diferentes do app:

1. **`checkForUpdatesOnStartup()`** — roda no fim de `loadAll()`, depois de
   tudo o resto (dados locais, sincronização, easter eggs) já ter
   carregado, para não atrasar nada. Busca a própria página (bypass de
   cache) e lê a constante `APP_VERSION` publicada via regex — a mesma
   técnica do botão manual "Procurar atualização" no Modo Dev, mas sem
   depender dele.
   - Se a versão remota for igual à local, não faz nada.
   - Se for diferente, grava essa versão em `pendingUpdateNotice`
     (device-local, via `window.storage`) e recarrega a página
     imediatamente com `location.replace(...)` — sem `confirm()`, sem
     sheet, sem esperar por nenhuma ação do utilizador.
   - Se a rede falhar ou a página não responder, falha silenciosamente e
     tenta de novo na próxima abertura.

2. **`consumePendingUpdateNotice()`** — chamada logo no início da
   abertura seguinte, já com a página recarregada (portanto já na versão
   nova). Consome e apaga a marca `pendingUpdateNotice`, e devolve
   `true`/`false` conforme a versão atual bater ou não com a marcada —
   não mostra nenhuma UI diretamente, quem chama decide onde exibir o
   aviso.

### 9.1 Por que o aviso vive na splash, não num toast separado

A primeira versão deste recurso mostrava um `showToast()` separado depois
de a tela de carregamento (splash) desaparecer. Na prática, isso nunca
era visto: a splash tem `z-index:999` (mais alto que o toast,
`z-index:100`) e o seu fade-out demora 400ms — como o toast era disparado
quase no mesmo instante em que a splash começava a desaparecer, ficava
tapado pela splash durante boa parte da sua própria animação de entrada.

A correção **unifica as duas telas**: em vez de um toast concorrendo com a
splash pela mesma janela de tempo, `loadAll()` verifica
`consumePendingUpdateNotice()` **antes** de esconder a splash e, se
verdadeiro, reaproveita a própria splash para mostrar a mensagem:

```js
const updateApplied = await consumePendingUpdateNotice();
if(updateApplied){
  setSplash(`✅ Atualizado para v${APP_VERSION}`);
  await new Promise(r => setTimeout(r, 1100));
}
hideSplash();
```

Ou seja, a splash troca o texto de "A sincronizar…" para "✅ Atualizado
para vX.X.X", fica visível mais 1.1s, e só depois começa a desvanecer —
garantindo que o aviso é sempre visto, porque não há mais duas telas
disputando o mesmo espaço e o mesmo instante.

## 10. Alterações desta versão (v3.1.0 → v4.0.0)

- **Logo da tela de carregamento**: corrigido para ser byte-idêntico ao
  ícone real do app (favicon/manifest/apple-touch-icon) — antes usava uma
  cor de fundo ligeiramente diferente (`#2C4F3D` em vez de `#1F3A2E`).
- **Easter egg "hackeamento"**: agora desativado por padrão em qualquer
  perfil/dispositivo novo — só liga com ativação explícita no Modo Dev.
- **Mensagem de aniversário**: agora desativada por padrão — só dispara
  depois de configurar uma data explicitamente; deixou de usar 19/10 como
  data implícita quando nada estava configurado.
- **Sincronização "pull antes de push"**: antes de qualquer gravação na
  nuvem, o app busca e mescla o estado mais recente da planilha, para
  editar sempre sobre a versão mais nova em vez de arriscar sobrescrever
  trabalho feito noutro dispositivo. Ver secção 3.
- **Documentação**: os grandes blocos de comentário explicativo em
  português foram removidos do `corridaplus.html` e movidos para este
  ficheiro, deixando o HTML principal mais leve.
- **Verificação automática de atualização**: o app agora se atualiza
  sozinho quando encontra uma versão nova publicada, sem perguntar nada —
  o aviso de "atualizado" só aparece depois, na abertura seguinte. Ver
  secção 9.
- **Correção: ganho/despesa apagado voltava sozinho**: o merge de
  pré-push estava a reintroduzir itens apagados localmente (porque ainda
  existiam na planilha remota no momento do merge). Corrigido com um
  sistema de lápides de exclusão (`deletedIds`) que impede o merge de
  trazer de volta algo apagado neste dispositivo. "Apagar todos os dados"
  também foi corrigido para não sofrer do mesmo problema. Ver secção 3.1.
- **Editar despesa variável**: as despesas variáveis (Despesas → Despesas
  variáveis) agora têm um botão de editar (✏️) além do de apagar (✕). Abre
  o mesmo sheet de criação, já preenchido, e grava por cima do registo
  existente (mantém o mesmo `id`) em vez de apagar e recriar. Se a data for
  alterada para outro mês, a despesa move-se automaticamente para o mês
  correto. `openExpenseSheet(existingExp)` decide entre modo "nova" e
  "editar" conforme recebe ou não um objeto de despesa existente;
  `openNewExpenseSheet()` continua a existir como atalho para o modo
  "nova".
- **Editar valor de um ganho diretamente na lista**: o valor de cada ganho
  em Ganhos agora é um campo editável inline (mesmo padrão já usado nas
  despesas fixas), em vez de só texto estático. `updateEarningAmount(id,
  value)` procura o ganho em todos os meses (a lista pode atravessar
  fronteira de mês na visão Semana) e grava o novo valor.
- **Correção: aviso de "app atualizado" nunca aparecia**: o toast
  ficava tapado pela tela de carregamento (splash), que tem `z-index`
  mais alto e ainda estava a desvanecer no momento em que o toast era
  disparado. As duas telas foram unificadas — a splash agora mostra
  diretamente "✅ Atualizado para vX.X.X" antes de desaparecer, em vez de
  um toast concorrente. Ver secção 9.1.
- **Número da versão sempre visível na splash**: adicionado um elemento
  fixo (`#splashVersion`) abaixo da mensagem de estado, mostrando sempre
  `vX.X.X` — independente do texto dinâmico ("A carregar…", "A
  sincronizar…", "Atualizado…") que ocupa `#splashMsg`. É definido de
  forma síncrona logo após a declaração de `APP_VERSION`, sem esperar por
  `loadAll()`, para aparecer desde o primeiro instante.
- **Metas semanal/mensal**: nova secção "Meta" em Você → interruptor
  "Ativar metas" + campos de meta mensal e meta semanal (€), guardados em
  `profile.goalsEnabled`/`monthlyGoal`/`weeklyGoal` (sincronizados). Quando
  ativo, Estatísticas → Mês e Estatísticas → Semana passam a mostrar um
  cartão de progresso (`goalProgressHTML()`) comparando o lucro líquido do
  período em curso com a meta definida — barra de progresso, percentagem,
  e quanto falta (ou "Meta atingida! 🎉"). Uma meta em branco/0 simplesmente
  não mostra o respetivo gráfico, mesmo com as metas ativadas.
- **Meta movida para o topo de Estatísticas**: o cartão de progresso da
  meta (mensal e semanal) agora aparece em primeiro lugar em
  `renderStatsMonth()`/`renderStatsWeek()`, antes dos cartões TVDE/Resumo,
  em vez de depois deles.
- **Meta mínima = despesas fixas, quando nenhuma meta é definida**: se
  `profile.monthlyGoal`/`weeklyGoal` estiver vazio ou 0, o gráfico de meta
  passa a usar as despesas fixas do período como meta mínima (cobrir os
  custos fixos), em vez de simplesmente não mostrar nada. O rótulo muda
  para "Meta mensal/semanal (mínimo — despesas fixas)" para deixar claro
  que não é uma meta definida manualmente. Para a visão Semana, isto exigiu
  que `totalFixedBills(entry)` passasse a aceitar um segundo parâmetro
  opcional `monthKeyOverride` — sem ele, continua a usar `currentMonthKey`
  como sempre (nenhum chamador existente precisou de mudar); com ele,
  calcula despesas fixas para o mês em que a semana em exibição começa
  (que pode não ser o mesmo mês de `currentMonthKey`), dividido pelo nº de
  semanas desse mês. Se não houver despesas fixas cadastradas, a meta
  mínima é 0 e o gráfico simplesmente não aparece (mesma regra de sempre
  em `goalProgressHTML()`).
- **Correção: despesas fixas não entravam no lucro semanal**:
  `renderStatsWeek()` (Estatísticas → Semana) e `renderLucroWeek()` (Lucro
  → Semana) calculavam o lucro da semana sem descontar nenhuma parcela das
  despesas fixas (bills são mensais, e as visões semanais simplesmente as
  ignoravam) — ao contrário das visões Mês, que sempre descontaram. Agora
  ambas calculam `fixasSemana` = despesas fixas do mês em que a semana
  começa, dividido pelo nº de semanas desse mês (mesmo rateio já usado nos
  gráficos "por semana" dentro da visão Mês) e descontam esse valor do
  lucro. Também passou a aparecer como item de lista ("🏠 Despesas fixas
  (rateio)") no card de Estatísticas → Semana. Isto também corrigiu uma
  inconsistência: antes, "Lucro da semana" podia mostrar valores
  diferentes em Estatísticas vs. Lucro para a mesma semana.
- **Metas renomeadas para "meta de lucro"**: título da secção nas
  Configurações, título do interruptor, rótulos dos campos, e os títulos
  dos cartões de progresso em Estatísticas passaram a dizer explicitamente
  "meta de lucro" (mensal/semanal), em vez de só "meta" — para não dar a
  entender que se trata de uma meta de faturamento bruto.
- **Formato de armazenamento alternativo ("schema v1"), opt-in no Modo
  Dev**: novo interruptor "Gravar no formato novo" que reestrutura o JSON
  enviado à planilha (separar configurações/brincadeiras do perfil,
  achatar ganhos/despesas em listas simples). Ver secção 11 para a
  arquitetura completa. Desativado por padrão — nada muda até o
  utilizador ativar explicitamente.

## 11. Formato de armazenamento alternativo (schema v1)

A partir da v3.6.0, existe um formato de armazenamento alternativo,
opt-in, ativável em Modo Dev → "Formato de armazenamento". Resolve as
críticas estruturais ao formato clássico (ver nota histórica abaixo) **sem
tocar em nenhuma outra parte da app** — a conversão acontece só na
fronteira da sincronização.

### 11.1 Nota histórica — críticas ao formato clássico

O formato original (`{bills, monthData, profile}`) tem três problemas
estruturais:
1. `profile` mistura identidade (nome/foto), configuração financeira
   (IVA/comissão/metas) e brincadeiras (easter egg/aniversário) no mesmo
   objeto plano, sem separação de responsabilidades.
2. `monthData` particiona ganhos/despesas por mês usando um objeto
   aninhado — mas a app já precisa achatar tudo de volta constantemente
   (`allEarnings()`, `earningsInRange()`) sempre que quer ver dados fora
   de um mês, então a partição não poupa trabalho, só acrescenta
   aninhamento.
3. Sem `schemaVersion` nem `updatedAt` — migrações são feitas com
   checagens ad-hoc espalhadas pelo código (`if(!earning.platform)
   earning.platform = 'uber'`), em vez de um número de versão central.

### 11.2 O formato novo (schema v1)

```json
{
  "schemaVersion": 1,
  "updatedAt": "2026-08-15T10:42:00Z",
  "profile": { "name": "...", "photo": "data:..." },
  "settings": {
    "ivaRate": 6, "comissaoRate": 4,
    "goals": { "enabled": true, "monthly": 1200, "weekly": 300 }
  },
  "features": {
    "easterEgg": { "disabled": true, "rate": 0.03 },
    "birthday": { "date": "19-10" }
  },
  "bills": [ { "id": "...", "name": "Água/Luz", "defaultAmount": 60, "frequency": "mensal", "createdMonthKey": "2026-07" } ],
  "earnings": [ { "id": "...", "date": "2026-07-28", "platform": "uber", "amount": 47.8 } ],
  "expenses": [ { "id": "...", "category": "combustivel", "amount": 30, "date": "2026-07-28" } ],
  "billOverrides": [ { "billId": "...", "month": "2026-07", "amount": 55 } ],
  "billsPaid": [ { "billId": "...", "month": "2026-07", "amount": 55, "paidAt": 1234567890 } ],
  "hiddenBills": [ { "billId": "...", "month": "2026-07" } ]
}
```

`earnings`/`expenses` deixam de estar aninhados por mês — passam a ser
duas listas simples com campo `date`; o agrupamento por mês/semana
continua a ser feito só no cliente (exatamente como já era, via
`earningsInRange()`). `profile`/`settings`/`features` ficam separados por
responsabilidade.

### 11.3 Como funciona por baixo dos panos

A **totalidade do resto da app continua a trabalhar só com o formato
clássico** (`bills`, `monthData`, `profile`) — nenhuma função de
renderização, cálculo, ou edição foi alterada. A conversão acontece só em
dois pontos:

- **`normalizeToSchemaV1(payload)`** — clássico → schema v1. Usado em
  `pushToCloud()` só quando `useNormalizedSchema` (device-local, Modo Dev)
  está ativo.
- **`denormalizeFromSchemaV1(data)`** — schema v1 → clássico. Usado em
  `resolveIncomingPayload(raw)`, um ponto único de deteção de formato
  (`raw.schemaVersion === 1` → converte; senão devolve como veio) chamado
  em **todos** os pontos onde dados entram na app: `finishCloudSync()`
  (cobre sincronização normal, decifrada automaticamente, e desbloqueio
  manual — os três convergem nessa função), `pullLatestAndMergeBeforePush()`,
  e `importFromJson()`.

Como a leitura entende os dois formatos **sempre, em qualquer
dispositivo**, é seguro ativar o interruptor num dispositivo só, sem
coordenar com os outros — cada dispositivo decide sozinho em que formato
escreve, e todos conseguem ler o que os outros escreverem. A conversão é
comprovadamente sem perdas (testada com round-trip: clássico → schema v1
→ clássico produz um objeto idêntico ao original).

### 11.4 Botão "Converter e enviar agora"

Além do interruptor (que só afeta gravações futuras), há um botão que
ativa o formato novo e força um envio imediato — útil para quem quer ver
o resultado na planilha sem esperar pela próxima edição.

## 12. Revertido: rateio de despesas fixas na visão Semana (v3.6.1)

A v3.5.1 tinha introduzido um rateio das despesas fixas mensais nas visões
Semana (Estatísticas e Lucro), dividindo o total do mês pelo nº de
semanas e descontando essa fatia do lucro semanal — para a semana nunca
mostrar um lucro "artificialmente alto" por ignorar despesas fixas por
completo.

Essa decisão foi revertida a pedido: a visão Semana volta a mostrar **só
o que aconteceu de facto nela** — sem inventar/dividir despesas de um
período diferente (o mês). `renderStatsWeek()` e `renderLucroWeek()`
voltaram ao cálculo original (fixas = 0 nas visões semanais). A meta
semanal também deixou de ter um "mínimo automático" baseado em despesas
fixas (que dependia do mesmo rateio) — sem uma meta semanal definida
explicitamente nas Configurações, o gráfico de meta semanal simplesmente
não aparece. A meta **mensal** mantém o seu mínimo automático (despesas
fixas do mês), porque aí não há rateio nenhum — é o valor real e completo
do próprio mês.

## 13. Correção: despesas diárias/semanais devem contar na visão Semana (v3.6.2)

A secção 12 acima foi longe demais: ao remover o *rateio* de despesas
**mensais**, a v3.6.1 acabou por remover **todas** as despesas fixas da
visão Semana — incluindo bills com frequência `diaria` e `semanal`, que
não são inventadas/divididas nenhuma: uma bill diária já tem um valor por
dia, e uma bill semanal já tem um valor por semana. Contá-las na visão
Semana não é rateio, é só somar o valor real pelo período certo.

Correção: `weeklyFixedBills(weekStart)` (nova função) calcula despesas
fixas apropriadas para uma semana, com uma regra clara:

- **`frequência: 'semanal'`** → conta o valor cheio (× 1) — já é o valor
  da semana.
- **`frequência: 'diaria'`** → conta o valor × 7 — sete dias de despesa
  diária real, não uma invenção.
- **`frequência: 'mensal'`** → **não conta** — incluir aqui exigiria
  dividir um valor pensado para o mês inteiro (isso sim seria o rateio
  indesejado). Essas despesas continuam a aparecer normalmente na visão
  Mês, no valor total.

Usada em `renderStatsWeek()` (mostrada como "🏠 Despesas fixas
(diárias/semanais)") e em `renderLucroWeek()` (via o parâmetro `fixas` de
`lucroBreakdownHTML()`). A meta semanal mínima automática também passou a
usar este valor real (em vez de ficar sem mínimo nenhum, como na v3.6.1).

### 13.1 Visão Mês: mensal e semanais juntos, sem rateio (v3.6.3)

Os gráficos "por semana" dentro de Estatísticas → Mês e Lucro → Mês ainda
usavam o rateio antigo (`fixasMes / nº de semanas`). Passaram a usar
`weekBucketFixedBills(mês, semana)`: cada semana do mês (blocos de dias
1–7, 8–14…) leva só as despesas fixas **diárias** (× dias reais do bloco)
e **semanais** (× 1). As **mensais** aparecem uma única vez, à parte.

Estatísticas → Mês ganhou o cartão "Lucro por semana e do mês": lucro de
cada semana, linha "Despesas mensais", e o "Lucro do mês" — as semanas
mais as despesas mensais somam exatamente o lucro do mês (soma dos blocos
+ mensais = `totalFixedBills()`, verificado).

## 14. Correção: despesa diária conta só nos dias com registro (v3.6.4)

A v3.6.2/3.6.3 multiplicava uma bill `diaria` por 7 (semana) ou pelos dias
do bloco (mês) — mas uma despesa diária representa o custo **daquele dia
específico**, não um valor fixo repetido todo santo dia. Se o motorista só
trabalhou 4 dias numa semana, só esses 4 dias têm a despesa diária.

Correção: `recordDays(entry)` reúne as datas que têm pelo menos um ganho ou
despesa variável lançados; `dailyBillsPerDay()` soma as bills diárias
aplicáveis nUM dia; e as três funções de despesas fixas passaram a contar
a diária **só nos dias com registro**, em vez de multiplicar por um número
fixo de dias:

- `totalFixedBills()` (mês/ano): diária × nº de dias com registro no mês
  inteiro.
- `weekBucketFixedBills()` (semanas dentro do mês, usado nos gráficos de
  Estatísticas/Lucro → Mês): diária × dias com registro dentro daquele
  bloco de 7 dias.
- `weeklyFixedBills()` (visão Semana isolada): olha cada um dos 7 dias da
  semana, no mês a que cada dia pertence (cobre semanas que atravessam a
  fronteira de mês), e soma a diária só nos dias com registro nesse mês.

A bill `semanal` continua a contar o valor cheio (× 1 numa semana, × nº de
semanas no mês) — não muda, porque já era exatamente "o valor da semana",
sem necessidade de olhar dia a dia. A soma dos blocos semanais + despesas
mensais continua a bater exatamente com o total do mês (testado).

## 15. Correção: despesa diária conta só em dias com DESPESA, não com ganho (v3.6.5)

A v3.6.4 contava a despesa fixa diária em qualquer dia com "algum
registro" — ganho OU despesa. Isso incluía dias em que só houve ganho
(trabalhou, mas não lançou nenhuma despesa variável nesse dia), o que não
faz sentido: trabalhar 5 dias mas só lançar combustível em 4 deles deve
gerar despesa diária de 4 dias, não 5.

Correção: `recordDays()` foi substituída por `expenseRecordDays()`, que
olha **só** `variableExpenses` (nunca `earnings`). As três funções de
despesas fixas (`totalFixedBills`, `weekBucketFixedBills`,
`weeklyFixedBills`) passaram a usar essa versão — a despesa diária conta
exclusivamente nos dias em que há pelo menos uma despesa variável
lançada nessa data, independente de ter havido ganho ou não nesse dia.

## 16. Opção por despesa: como a despesa MENSAL aparece nas semanas (v3.7.0)

Nova opção, configurável individualmente em cada despesa fixa com
frequência `mensal` (tanto ao criar quanto depois, editando), controlando
como ela aparece nas visões "por semana":

- **Não aparecer nas semanas (padrão)** — mantém o modelo anterior: a
  despesa só é contada na visão Mês, no valor total.
- **Dividir entre as semanas do mês** — reparte o valor em partes iguais
  por todas as semanas do mês (o "rateio" que tinha sido removido
  globalmente antes, agora disponível como escolha explícita por despesa).
- **Colocar tudo na 1ª semana do mês** — lança o valor inteiro de uma vez
  na primeira semana (útil para despesas que realmente são pagas logo no
  início do mês, como um aluguel).

Guardado em `bill.weeklySplit` (`'none'` | `'rateio'` | `'firstWeek'`,
padrão `'none'` quando ausente, para despesas já existentes continuarem
com o comportamento de sempre). Só é relevante para `frequency: 'mensal'`
— despesas diárias/semanais já têm um valor natural por semana e não usam
este campo.

`totalFixedBills()` (total do mês) nunca muda — a despesa mensal continua
a valer o valor cheio, uma vez, independente da escolha. O que muda é só
como `weekBucketFixedBills()` (dentro da visão Mês) e `weeklyFixedBills()`
(visão Semana isolada) repartem esse mesmo valor: a soma das semanas +
"despesas mensais" restantes continua sempre a bater com o total do mês,
qualquer que seja a combinação de escolhas entre as despesas (testado).

UI: seletor no sheet "Nova despesa fixa" (só aparece quando a frequência
escolhida é Mensal) e um `<select>` inline em cada despesa mensal já
existente na lista de Despesas fixas.

## 17. Firebase — login Google + Firestore (v4.0.0)

Mudança arquitetural: o app passou a exigir login com conta Google, e a
sincronização principal passou do Google Apps Script para o Firestore.
Detalhe técnico completo em `corridaplus-firebase-plan.md`. Resumo:

- `<script type="module">` novo no `<head>` inicializa Firebase Auth +
  Firestore e expõe `window.fbSignInWithGoogle`, `fbSignOut`,
  `fbSaveUserData`, `fbLoadUserData`, `fbListenUserData`,
  `fbAuthReadyPromise` para o script clássico usar.
- `loadAll()` agora espera por `fbAuthReadyPromise` antes de tudo — sem
  sessão, mostra `#loginGate` e para; com sessão, continua.
- Um documento por utilizador (`users/{uid}`), no mesmo formato "schema
  v1" já existente — reaproveita `normalizeToSchemaV1()`/
  `resolveIncomingPayload()` sem alterações.
- Migração automática no primeiro login (envia dados locais existentes
  se a conta ainda não tiver nada na nuvem) e sincronização em tempo
  real via `onSnapshot()`.
- Apps Script mantido no código como reserva (não deve ser alcançado na
  prática, já que o login passou a ser obrigatório).
- **Pendente**: regras de segurança do Firestore ainda precisam de ser
  coladas manualmente no console (sem elas, leitura/escrita falha).
