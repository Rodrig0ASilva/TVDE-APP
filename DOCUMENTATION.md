# Corrida+ — Documentação técnica

Este ficheiro reúne a documentação de arquitetura e o histórico de alterações da app `corridaplus.html`.
O HTML principal mantém apenas comentários curtos, apontando para aqui quando for preciso mais contexto.

**Versão atual: 1.0.2a.** A última secção deste documento é a versão atual.

## Índice
- **Parte I — Documentação técnica**: sincronização, IVA e lucro, encriptação, biometria, atualização, armazenamento, Firebase e funcionalidades específicas (secções 1 a 24).
- **Parte II — Histórico por versão**: uma secção por versão, da 0.4.2 até à 0.80 (a atual).

---

# Parte I — Documentação técnica

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

> **Nota (v0.4.11):** a união por ID e as lápides descritas nas secções 3 e
> 3.1 foram substituídas pela fusão com base em três vias (ver a entrada da
> versão 0.4.11 na Parte II (histórico por versão)). A secção 3 fica como histórico.

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

> **Numeração:** a partir da versão 0.46, a numeração passa a ser 0.46, 0.47, 0.48, … As entradas anteriores mantêm a numeração 0.4.x.

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

## 12. Revertido: rateio de despesas fixas na visão Semana
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

## 13. Correção: despesas diárias/semanais devem contar na visão Semana
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

## 14. Correção: despesa diária conta só nos dias com registro
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

## 15. Correção: despesa diária conta só em dias com DESPESA, não com ganho
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

## 16. Opção por despesa: como a despesa MENSAL aparece nas semanas
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

## 17. Firebase — login Google + Firestore
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

## 18. Login Google voltou a ser opcional, por enquanto
O login obrigatório da v4.0.0 bloqueava o app inteiro se o
`signInWithPopup` falhasse (ex: domínio não autorizado nas definições do
Firebase Auth, popup bloqueado pelo browser) — impossível de testar o
resto da app enquanto isso não estivesse resolvido.

Mudança: `loadAll()` deixou de esperar/bloquear em
`window.fbAuthReadyPromise` — só lê o estado (`window.__fbUser`) e segue
em frente de qualquer forma. `showLoginGate()` continua definida mas já
não é chamada automaticamente. O botão "Continuar com Google" mudou-se
para Configurações → Conta (`#accountCardSignedOut`), onde fica
disponível para quem quiser ligar a conta, sem ser obrigatório. Sem
sessão, a app funciona exactamente como antes do Firebase (Apps
Script/local) — a lógica de fallback em `scheduleCloudPush()`/`loadAll()`
já tratava isto corretamente, só não era alcançada por causa do bloqueio.

`handleGoogleSignIn()` passou a atualizar qualquer um dos dois pares
botão/erro presentes na página (o do ecrã de login, não usado por
agora, e o novo em Configurações), para continuar a funcionar nos dois
sítios sem duplicar código.

Quando o login com Google estiver a funcionar de forma fiável, o bloqueio
obrigatório pode voltar a ser ligado facilmente — é só restaurar a
verificação no início de `loadAll()`.

## 19. Esquema de versão voltou a pré-1.0
A numeração 4.x.x usada temporariamente durante a integração Firebase foi
abandonada. De volta a 0.x.x — a v1.0.0 fica reservada para o lançamento
com o recurso completo de assinatura paga (ver `corridaplus-firebase-plan.md`,
Fase 3).

## 20. Turno de trabalho (opcional) + gráfico "Ganhos por turno"
Novo campo opcional no sheet "Novo ganho": um seletor de turno (Manhã,
Tarde, Noite, Madrugada, ou "Não dizer") — `TURNOS` define as opções.
Guardado como `earning.turno` em cada ganho criado nessa submissão
(`confirmAddIncome()`), só quando o utilizador escolhe um turno
explicitamente.

`turnoBreakdownHTML(items)` gera um gráfico de barras "Ganhos por turno"
com o total ganho em cada turno e destaque do melhor — **só aparece se
houver pelo menos um ganho com turno preenchido** no período em questão
(se ninguém preencher, a função devolve string vazia e nada é mostrado).
Integrado em todas as visões de Estatísticas: Dia, Semana, Mês e Ano.

## 21. Plataformas personalizadas (opcional)
Além de Uber e Bolt (TVDE) e Particular/Outros (pessoal), o utilizador
pode agora adicionar outras plataformas (ex: 99, InDrive, Free Now) em
Configurações → "Outras plataformas". Guardadas em
`profile.customPlatforms: [{key, label}]` (sincronizado como parte do
profile, igual a qualquer outra configuração).

- `addCustomPlatform()` gera uma `key` normalizada a partir do nome (ex:
  "Free Now" → `custom_free_now`) e adiciona à lista.
- `removeCustomPlatform(key)` remove da lista de opções futuras — ganhos
  já registados com essa plataforma continuam guardados e visíveis, só
  deixa de aparecer como opção ao criar um novo ganho.
- `openAddIncomeSheet()`/`confirmAddIncome()` passaram a gerar um campo
  de valor extra por cada plataforma personalizada ativa, dinamicamente.
- `platformBadge()`/`platformLabel()` generalizados com
  `findCustomPlatform(platform)` para reconhecer e exibir as plataformas
  personalizadas (emoji/inicial + nome) em qualquer lista/card existente,
  sem precisar de tratamento especial em cada sítio.
- **Regra fiscal**: toda plataforma personalizada é tratada como
  "pessoal" (mesmo balde que Particular/Outros) — sem desconto de
  IVA/comissão, porque essa lógica é especificamente modelada para a
  comissão de apps TVDE (Uber/Bolt) e não generaliza sem pedir ao
  utilizador plataforma a plataforma. Nova função `isTvdePlatform(platform)`
  (só verdadeira para `'uber'`/`'bolt'`) substitui, em 11 sítios do
  código, as antigas verificações explícitas `platform==='particular' ||
  platform==='outros'` — qualquer plataforma personalizada passa a entrar
  automaticamente nesse balde sem precisar de mais nenhuma alteração.
- Cards de "Total bruto"/"Pessoal" em Ganhos, Estatísticas → Mês e
  Estatísticas → Ano atualizados para incluir e discriminar os totais de
  cada plataforma personalizada.

## 22. Pacote de ajustes
- **Schema v1 ativo por padrão**: `useNormalizedSchema` passou a `true`
  por padrão (tanto o valor inicial em memória quanto o fallback quando
  não há preferência guardada em `window.storage`). Dispositivos que já
  tinham desativado explicitamente continuam respeitando essa escolha.
- **Líquido abaixo do bruto em Ganhos**: cada linha da lista de ganhos
  (`renderEarningsList()`) agora mostra "líq. €X" em fonte pequena por
  baixo do valor editável, só para plataformas TVDE (Uber/Bolt) e só
  quando há IVA ou comissão configurados — para plataformas pessoais
  líquido = bruto, então a linha extra seria redundante e seria omitida.
- **Correção visual dos campos de "Novo ganho"**: os 4 campos fixos
  (Uber, Bolt, Particular, Outros) e os campos de plataformas
  personalizadas estavam dentro de `<div style="...">` em vez de
  `<div class="field" style="...">` — como a regra de modo escuro/tema
  (`body.dark .field input`) depende da classe `.field` como ancestral,
  esses inputs específicos ficavam com a aparência padrão do navegador em
  vez de respeitar o tema. Corrigido adicionando a classe em falta.
- **Exportação para Excel**: nova secção em Configurações →
  "Exportação de dados", com o botão "Exportar para Excel". Gera um
  `.xlsx` com 4 folhas — Ganhos (todos, com IVA/comissão/líquido
  discriminados e turno se preenchido), Despesas, Despesas Fixas, e um
  Resumo Mensal (bruto TVDE/pessoal, IVA, comissão, despesas, lucro) —
  pensado para servir de base à declaração de IRS. Usa a biblioteca
  SheetJS, carregada via CDN (`cdnjs.cloudflare.com`) só para esta
  funcionalidade; o ficheiro é gerado inteiramente no dispositivo, nada é
  enviado para fora.

## 23. Reforço de segurança: App Check + regras do Firestore com validação de schema
Em vez de tentar "esconder" o JS (impossível — ver conversa), o reforço
real foi investido em duas frentes, ambas detalhadas em
`corridaplus-firebase-plan.md`:

- **Firebase App Check** (reCAPTCHA v3) — bloco novo no script-módulo
  Firebase, condicional a `APP_CHECK_SITE_KEY` ser substituído por uma
  chave real (placeholder por padrão = desativado, não bloqueia nada
  enquanto não for configurado). Garante que pedidos ao Firestore vêm
  mesmo desta app, não de um clone do código a imitar os pedidos.
- **Regras do Firestore reforçadas**: além de verificar "é dono do
  documento" (`isOwner`), passaram a validar também a FORMA dos dados
  (`hasValidShape()` — `request.resource.data.keys().hasOnly([...])`),
  impedindo um cliente alterado de injetar campos fora do schema v1
  esperado (ex: um campo fake de assinatura ativa escrito diretamente no
  próprio documento do utilizador).
- **Desenho correto do gate de assinatura (Fase 3)**: documentado para
  nunca confiar num campo dentro do documento que o próprio cliente pode
  escrever — a validação de assinatura ativa vai sempre consultar a
  coleção `customers/{uid}/subscriptions`, escrita exclusivamente pela
  extensão Stripe via Cloud Functions (que ignoram as regras do cliente).

## 24. Cartão "Assinatura" em Configurações, com data de validade
Novo cartão em Configurações → "Assinatura", logo abaixo de "Conta",
mostrando o estado da assinatura e até quando está ativa — já preparado
para a Fase 3 (Stripe), mesmo antes de ela estar configurada.

- `fbListenSubscription(uid, callback)` (módulo Firebase) lê
  `customers/{uid}/subscriptions` filtrando por `status in
  ['active','trialing']` — essa coleção só é escrita pela extensão
  Stripe (Cloud Functions, nunca pelo cliente). Se a coleção ainda não
  existir ou estiver vazia, devolve `null` sem erro.
- `window.__fbSubscription` tem três estados possíveis:
  `undefined` (a verificar), `null` (verificado, sem assinatura — normal
  antes do Stripe estar configurado), ou o documento da assinatura.
- `renderSubscriptionCard()` traduz isso em texto: "A verificar…", "🔓 Sem
  assinatura ativa" (com nota de que a cobrança ainda não está
  configurada), ou "✅ Assinatura ativa" / "🎁 Período de teste" +
  "Ativa até DD de mês de AAAA" (lido de `current_period_end`, aceitando
  tanto Timestamp do Firestore quanto string/número, conforme a versão
  da extensão). Se `cancel_at_period_end` estiver marcado, mostra
  "Termina em [data] (cancelamento agendado)" em vez de "Ativa até".


# Parte II — Histórico por versão

Histórico agrupado por assunto, em ordem cronológica. A última secção é a versão atual (0.80).

## Versões 0.4.2 a 0.4.5 — Base: turnos, plataformas extra, Excel, segurança e cartão de assinatura

- **Versão 0.4.2 — esquema de versão pré-1.0, turno de trabalho e plataformas extra**: a numeração
  voltou a 0.x.x (secção 19). Turno de trabalho opcional e gráfico "Ganhos por turno" (secção 20).
  Plataformas personalizadas, como 99 ou InDrive (secção 21).

- **Versão 0.4.3 — pacote de ajustes**: schema v1 ativo por defeito; líquido nas linhas de ganhos
  (TVDE); correção visual dos campos de "Novo ganho"; exportação para Excel (secção 22).

- **Versão 0.4.4 — reforço de segurança**: App Check e regras do Firestore com validação de schema
  (secção 23).

- **Versão 0.4.5 — cartão "Assinatura"**: estado da assinatura e data de validade em Configurações,
  preparado para a Fase 3 (secção 24).

## Versão 0.4.6 — Modo Dev, sincronização e ajustes de configurações

- **Versão 0.4.6 — botão GitHub no Modo Dev**: o link "Ver no GitHub" saiu do
  ecrã "Você" e foi para o Modo Dev, com o novo endereço
  `https://github.com/Rodrig0ASilva`.

- **Versão 0.4.6 — sincronização no Modo Dev**: a secção "Sincronização" (URL do
  Google Apps Script, bloquear/editar) foi movida do ecrã "Você" para dentro do
  Modo Dev.

- **Versão 0.4.6 — conta de suporte no Modo Dev**: mostra o email e o UID da
  conta Google, com botão "Copiar email + UID". O UID também está no Firebase
  Console → Authentication → Utilizadores, onde se pode procurar pelo email.

- **Versão 0.4.6 — "Apagar todos os dados" sempre no fim**: o botão fica depois
  do Modo Dev, como última opção do ecrã "Você".

- **Versão 0.4.6 — "Registos opcionais"**: nova secção em "Você" com interruptores
  para **Turno de trabalho**, **Quilómetros (km)** e **Usar outras plataformas**.
  Os três são guardados em `profile` (`showTurno`, `showKm`,
  `customPlatformsEnabled`) e sincronizam entre dispositivos. Por omissão, turno
  começa ligado, km desligado, e outras plataformas ligadas só se já existirem
  plataformas registadas. Desativar um interruptor esconde a funcionalidade, mas
  não apaga os dados já gravados.

- **Versão 0.4.6 — km nas Estatísticas**: o campo de km do ganho aparece no
  formulário "Novo ganho" quando o interruptor está ativo. Se for preenchido, as
  Estatísticas mostram o cartão "Quilometragem" (km registados e lucro por km) e
  o Lucro mostra o lucro por km. Os km do dia são gravados no primeiro ganho desse
  registo, para não serem contados em dobro.

- **Versão 0.4.6 — lista de outras plataformas**: a lista e o campo para adicionar
  plataformas só aparecem com o interruptor "Usar outras plataformas" ativo.

## Versões 0.4.7 a 0.4.9 — Perfil e conta Google

- **Versão 0.4.7 — perfil e conta num só cartão**: em Configurações → "Perfil e
  conta", a secção "Conta" separada foi integrada no cartão do perfil. A foto por
  omissão é a foto da conta Google (`currentPhotoUrl()`: foto própria, ou se não
  existir, a do Google). O botão "trocar" continua a permitir enviar outra foto, e
  "Repor foto do Google" volta à foto da conta. O nome é livre, com o nome do
  Google como sugestão no campo. Por baixo aparecem o email e o ID da conta
  (UID); sem sessão, aparece o botão "Continuar com Google". O botão "Sair da
  conta" também está neste cartão. A secção "Conta (suporte)" do Modo Dev foi
  removida.

- **Versão 0.4.8 — ajustes ao cartão de conta**: símbolo "G" do Google com o
  texto "Conectado com Google" acima do email. O botão "Sair" fica à direita, com
  fundo branco e letra vermelha. O ID da conta (UID) aparece mascarado como
  `****`, com um botão "mostrar"/"ocultar". A escolha fica guardada só neste
  dispositivo (`localSettings.showUid`), sem sincronizar.

- **Versão 0.4.9 — bloco "Conta"**: o estado da assinatura saiu da secção
  própria e passou para a parte inferior do cartão de conta, sob o título
  "Assinatura". A secção de Configurações passou a chamar-se apenas "Conta".

## Versões 0.4.10 a 0.4.11 — Sincronização: correções e fusão com base

- **Versão 0.4.10 — correções da Fase 1**:
  - Listeners do Firestore (dados e assinatura) são cancelados antes de voltar a
    ser registados. Tocar em "sincronizar" já não acumula listeners.
  - A brincadeira e a verificação de atualização só correm no primeiro
    carregamento da sessão, e não ao tocar em "sincronizar".
  - "Importar dados" agenda o envio para a nuvem, para os dados importados
    chegarem aos outros aparelhos.
  - Nomes de plataformas e categorias são escapados antes de entrar no HTML.
  - (Revertido na 0.4.12.) O gráfico diário de Lucro (Semana) passou a usar a
    mesma fórmula do resumo. A alteração foi revertida a pedido.
  - Remover uma plataforma extra passa a arquivá-la (`archived: true`). Deixa de
    aparecer como opção nova, mas os ganhos antigos mantêm o nome. Re-adicionar a
    mesma plataforma volta a ativá-la.

- **Versão 0.4.11 — Fase 2 da sincronização**:
  - **Fusão com base em três vias** (`mergeState`, `syncWithCloud`). A app guarda
    o último estado conhecido da nuvem (`syncBase`, por fonte: uma conta ou o
    Apps Script). Ao sincronizar, vence o lado que mudou desde a base; se os dois
    mudaram, vence o local, e campos diferentes do mesmo item fundem-se. Edições e
    exclusões de cada aparelho chegam aos outros, sem ressuscitar itens apagados.
    Isto substitui a união por ID e as lápides.
  - **Snapshot não substitui mais o estado local.** Quando chega uma alteração de
    outro aparelho, é fundida com a base. Edições ainda não enviadas ficam
    preservadas.
  - **Envio pela conta Firebase só depois de ler a nuvem.** `pushToFirestore()`
    lê o documento e funde antes de gravar. Se a leitura falhar, não grava.
  - **Primeiro acesso com dados diferentes**: quando a conta tem dados na nuvem e
    este aparelho também tem dados, a app pergunta. A folha mostra a data em que a
    nuvem foi guardada (`updatedAt`) e um resumo de cada lado (ganhos, despesas
    variáveis e fixas). As opções são "Usar os dados da nuvem", "Manter os deste
    aparelho (substitui a nuvem)" e "Decidir depois". Fechar a folha equivale a
    "Decidir depois": nada é enviado, e a pergunta volta a aparecer na próxima
    abertura ou ao tocar em sincronizar.
  - **Apagar todos os dados** substitui a nuvem (`pushToFirestore(true)`).
  - A fusão foi testada isoladamente com nove cenários de dois aparelhos
    (edição remota, exclusão local e remota, adições dos dois lados, conflito no
    mesmo item, campos diferentes do mesmo item, foto removida, e base vazia).
  - (Na 0.4.14, a pergunta sobre dados locais ou da nuvem foi removida: a nuvem prevalece.)
  - Limitação: a base guarda uma cópia completa dos dados no armazenamento local.
    Com muito histórico, isto pesa; quando o item 9 da lista (limite de 1 MiB no
    Firestore) for tratado, convém rever este formato.

## Versões 0.4.12 a 0.4.15 — IVA, comissão e plataformas com impostos

- **Versão 0.4.12 — reversão**: o gráfico diário de Lucro (Semana) volta à fórmula
  anterior, `(bruto − combustível − IVA sobre o bruto sem combustível) + pessoal −
  outras variáveis`, e à legenda "Lucro líquido de cada dia da semana". A
  alteração da 0.4.10 (item 14) foi revertida a pedido.

- **Versão 0.4.13 — Configurações agrupadas**: as secções "Registos opcionais",
  "Meta de lucro" e "Outras plataformas" passaram a ser um só grupo, "Opcionais",
  com os interruptores de turno, km, IVA/imposto, metas de lucro e outras
  plataformas. A comissão continua num grupo próprio, logo abaixo.
- **IVA / Imposto com chave de ativação** (`profile.ivaEnabled`). Desligado, `ivaRate()`
  devolve 0 e todos os cálculos e ecrãs que dependem do IVA deixam de o mostrar. A
  taxa continua guardada, para voltar a ativar sem a introduzir de novo. Por
  omissão, fica ativo quando já existe uma taxa maior que 0.

- **Versão 0.4.14 — ajustes à sincronização e ao grupo de opcionais**:
  - **Nuvem prevalece no primeiro acesso.** Quando a conta tem dados na nuvem, a
    app usa esses dados sem perguntar, mesmo que o aparelho tenha dados próprios.
    O aparelho só envia os seus dados quando a nuvem está vazia (utilizador novo).
    Isto substitui a escolha "manter os locais ou os da nuvem" da 0.4.11, que foi
    removida. A fusão com a base continua a ser usada nas sincronizações seguintes.
  - **Apagar todos os dados** exige um código aleatório de 8 caracteres, que tem de
    ser escrito antes de o botão ficar ativo. A folha mostra o aviso de que a ação
    é permanente, de que não há cópia de segurança, e de que a Corrida+ não se
    responsabiliza por perdas de dados.
  - **IVA / Imposto e Comissão da plataforma** ficam no topo do grupo "Opcionais",
    juntos e cada um com o seu seletor (`profile.ivaEnabled` e
    `profile.comissaoEnabled`). Desligado, o valor deixa de entrar nos cálculos e
    nos ecrãs, e a taxa fica guardada.

- **Versão 0.4.15 — plataformas com IVA e comissão**:
  - Ao adicionar uma plataforma extra, pergunta-se se cobra IVA e comissão. Por
    omissão, a resposta é "Não (pessoal)". Cada plataforma tem o campo
    `profile.customPlatforms[].taxed`.
  - Na lista de plataformas, o botão "pôr IVA"/"tirar IVA" altera essa escolha
    depois de criada. A alteração aplica-se a todos os ganhos dessa plataforma,
    incluindo os antigos, porque os valores são calculados na hora.
  - `isTvdePlatform()` passou a incluir as plataformas marcadas com IVA. Todas as
    comparações diretas com Uber/Bolt foram trocadas por essa função, para que IVA,
    comissão, líquido, lucro e exportação tratem as duas da mesma forma.
  - Nos ecrãs Ganhos (resumo), Estatísticas → Ano e Estatísticas → Mês, o bruto das
    plataformas com IVA entra no total TVDE. O cartão "Pessoal" não as mostra.
    Limitação: nas listas por plataforma das Estatísticas (Dia, Semana, Mês), essas
    plataformas ainda não têm cartão próprio.
  - O grupo de configurações "Opcionais" passou a chamar-se "Configurações".

## Versões 0.4.16 a 0.4.18 — Resumo de ontem, apagar e exportar

- **Versão 0.4.16 — resumo de ontem**: novo interruptor em Configurações →
  Configurações (`profile.dailySummaryEnabled`, desligado por omissão). Na primeira
  abertura do dia, mostra uma folha com o resumo do dia anterior: bruto TVDE,
  pessoal, combustível, outras despesas, km (se o interruptor de km estiver ligado)
  e lucro do dia. O resumo não inclui despesas fixas, como nas vistas de dia.
  - A data em que o resumo foi mostrado pela última vez fica em `localSettings`,
    que não sincroniza. Assim, cada aparelho mostra o resumo uma vez por dia.
  - Só é mostrado se existir algum registo no dia anterior. Se não houver, não
    marca o dia como mostrado.
  - Com sessão Firebase, o resumo é verificado depois da sincronização inicial,
    para usar os dados da nuvem.

- **Versão 0.4.17 — apagar, redefinir e exportar**:
  - **"Apagar todos os dados" passa a perguntar o que apagar**, com três opções:
    - *Apagar só os dados*: ganhos, despesas variáveis, despesas fixas e histórico.
      Mantém configurações, nome e foto.
    - *Redefinir só as configurações*: tema, IVA, comissão, metas, plataformas extra,
      opcionais, resumo de ontem, encriptação e biometria. Mantém dados, nome e foto.
    - *Apagar tudo*: dados, configurações, nome e foto.
    Cada opção pede o código aleatório de 8 caracteres e mostra os avisos de
    responsabilidade e de perda permanente.
  - A limpeza é enviada à nuvem com substituição (`pushToFirestore(true)` ou
    `pushToCloud(undefined, true)`), e a base de fusão é limpa. Assim, os dados
    apagados não voltam pela fusão.
  - A ligação à planilha e a conta Google não são apagadas por nenhuma das opções.
    São ligações, não preferências.
  - **Exportar** abre uma escolha entre:
    - *Planilha Excel (.xlsx)*, para a declaração de IRS (como antes);
    - *Arquivo de configurações (.json)*, uma cópia completa dos dados e das
      configurações no formato v1, que pode ser importada em Modo Dev → Importar
      dados. Não inclui a senha de encriptação.

- **Versão 0.4.18 — resumo de ontem no Modo Dev e com km e turno**:
  - Modo Dev → "📊 Resumo de ontem" → "Forçar resumo na próxima abertura". Ignora o
    interruptor e o registo de "já mostrado hoje", e mostra o resumo uma vez. Se o
    dia anterior não tiver registos, mostra o resumo vazio. Depois de mostrado, o
    botão volta ao normal (`devForceDailySummary`, não sincronizado).
  - O resumo mostra a linha de km sempre que o interruptor de km estiver ligado,
    mesmo quando o valor é 0.
  - O resumo mostra o gráfico "Ganhos por turno" de ontem quando o interruptor de
    turno estiver ligado. Se não houver turnos registados, mostra "Sem turno
    registado ontem".
  - Com sessão Firebase, a verificação do resumo corre no fim da sincronização
    inicial, mesmo quando ela termina antes de houver dados.

## Versões 0.4.19 a 0.4.22 — Ajustes de layout, detalhamento e versão

- **Versão 0.4.19 — alinhamentos em Configurações e botão de apagar**:
  - Alinhamentos: removida a margem negativa do texto das metas de lucro, que
    puxava o texto para cima do campo. Os campos do grupo "Configurações" usam a
    mesma margem superior (8px). O texto da lista de plataformas extra usa margem
    vertical igual aos outros textos da secção.
  - "Apagar dados ou configurações" fica separado do resto por uma linha tracejada
    em vermelho, com o título "Zona de perigo". O botão é vermelho e sólido, com
    sombra, e tem a legenda "Ação permanente. Pede um código de confirmação antes de
    apagar."
  - Versão do app: 0.4.19 (mostrada no rodapé de Configurações).

- **Versão 0.4.20 — ajustes de ganhos, detalhamento e turnos**:
  - **Ganhos**: removida a linha "líq. €" por baixo do valor de cada ganho.
  - **Lucro · Detalhamento**: o tracejado que ficava por baixo de "Comissão
    plataforma" foi removido. Fica só o tracejado por baixo de "Líquido TVDE".
  - **Turnos com nome configurável**: em Configurações → Turno de trabalho, quando o
    interruptor está ligado, aparecem quatro campos (um por turno), com máximo de 14
    letras. Em branco, usa o nome original. Os nomes são guardados em
    `profile.turnoNames`, que sincroniza entre aparelhos, e aplicam-se ao formulário de
    ganho, ao gráfico "Ganhos por turno" e à exportação para Excel.

- **Versão 0.4.21 — versão no fim de Configurações**: o rodapé "Corrida+ vX.X.X"
  passou para o fim do ecrã, abaixo da zona de perigo e do Modo Dev. Continua a
  abrir o Modo Dev com 5 toques.

- **Versão 0.4.22 — ajustes de detalhamento, versão e turnos**:
  - **Lucro · Detalhamento**: o tracejado por baixo de "Líquido TVDE" foi removido.
    Não fica nenhum tracejado nessa zona.
  - **Versão do app**: sai do fim de Configurações e passa para a aba Você, logo
    abaixo do estado de sincronização. Continua a abrir o Modo Dev com 5 toques.
  - **Turnos**: cada turno tem um campo de emoji e um campo de nome, em
    Configurações → Turno de trabalho. A lista mostra o nome e o emoji atuais. O
    emoji aceita um único símbolo. Os valores são guardados em `profile.turnoNames`
    e `profile.turnoEmojis`, e aplicam-se ao formulário de ganho, ao gráfico "Ganhos
    por turno" e à exportação.

## Versões 0.4.23 a 0.4.27 — Turnos editáveis, Modo Dev e reposição

- **Versão 0.4.23 — turnos editáveis (adicionar e remover)**:
  - Em Configurações → Turno de trabalho, a lista de turnos pode ser editada: mudar
    o nome e o emoji, remover um turno, reativar um turno removido, ou adicionar
    turnos novos (nome de até 14 letras e um emoji).
  - A lista fica em `profile.turnoList`, que sincroniza entre aparelhos. Sem lista
    guardada, aparecem os 4 turnos padrão, com os nomes e emojis personalizados que
    já existiam.
  - Remover um turno marca-o como `archived`. Os ganhos antigos continuam a mostrar
    o turno nos gráficos e na exportação, mas o turno deixa de aparecer no formulário
    de ganho. Pode ser reativado na lista de "Removidos".

- **Versão 0.4.24 — Modo Dev em aba própria e guardado**:
  - O Modo Dev deixou de ser uma secção no fim de Configurações. Passa a ser um
    ecrã próprio, com a aba "Dev" na barra inferior. As opções são as mesmas.
  - A aba só aparece com o Modo Dev ativo. Ativar é feito com 5 toques no número da
    versão. Desativar é feito pelo interruptor no topo do próprio ecrã.
  - O estado fica guardado em `localSettings.devMode`, no aparelho. Assim, o Modo
    Dev continua ativo depois de fechar e reabrir a app, até ser desativado. A
    reposição de configurações não o desativa.
  - Em Configurações, a secção de exportação já não menciona o Modo Dev. O arquivo
    .json continua disponível, sem indicação de onde é importado.

- **Versão 0.4.25 — correções de turnos, aba Dev e temas**:
  - **Turnos desapareciam ao remover ou adicionar.** A fusão com a base usava só o
    campo `id` para identificar itens de lista. Turnos e plataformas extra usam
    `key`, por isso eram descartados na fusão seguinte. Agora `idOf()` usa `id` ou
    `key` (e, sem nenhum dos dois, o próprio conteúdo). Testado com turnos
    removidos, adicionados e criados noutro aparelho, e com plataformas extra.
  - **Aba Dev** fica à direita de "Você", a última da barra.
  - **Apagar configurações** (redefinir ou apagar tudo) desativa o Modo Dev e
    esconde a aba Dev. Apagar só os dados mantém o Modo Dev.
  - **Temas**: os campos de nome e emoji dos turnos e as opções das folhas de
    exportação e de apagar seguem o modo claro/escuro (classes `theme-input` e
    `theme-card`).

- **Versão 0.4.26 — reposição volta ao padrão, incluindo turnos**:
  - Ao apagar configurações ou tudo, a base de sincronização passa a ser o estado
    já reposto. Antes ficava vazia, e a versão antiga da nuvem era tratada como
    primeira sincronização, o que restaurava as configurações antigas (turnos
    incluídos).
  - Durante a reposição, as alterações vindas da nuvem são ignoradas
    (`window._suppressRemote`), até o novo estado ser gravado.

- **Versão 0.4.27 — turnos padrão repostos em contas antigas**:
  - Contas antigas podiam ter a lista de turnos incompleta por causa do erro da
    fusão corrigido na 0.4.25. Quando um turno padrão faltava, a lista ficava sem
    ele, e apagar as configurações não o repunha.
  - `repairTurnoList()` garante que os 4 turnos padrão existem. Corre ao abrir a app,
    ao aplicar dados da nuvem e ao importar. Se algum faltar, é reposto e a alteração
    é gravada. Um turno removido pelo utilizador fica arquivado e não é reposto.
  - Em Configurações → Turno de trabalho, o botão "↺ Repor turnos padrão" volta aos
    4 padrões sem apagar mais nada. Os turnos personalizados ficam arquivados, para
    os ganhos antigos continuarem com o nome certo.
  - Turnos personalizados que tenham sido perdidos pelo erro antigo não se
    recuperam. Só voltam se houver uma cópia de segurança (arquivo .json).

## Versão 0.4.28 — Cores do tema no modo escuro

- **Versão 0.4.28 — cores do modo escuro seguem o tema**: o separador ativo da barra
  (e a opção selecionada nas plataformas e no seletor de mês) usava um amarelo fixo
  em modo escuro. Agora usa a cor de texto do tema escolhido, com um fundo neutro.

## Versões 0.4.29 a 0.4.32 — Boas-vindas, modo básico e tela de carregamento

- **Versão 0.4.29 — boas-vindas e modo básico**:
  - **Boas-vindas**: na primeira abertura sem sessão Google, aparece uma folha com
    "Entrar com Google" e "Continuar em modo básico". A escolha fica guardada em
    `localSettings.welcomeDone`.
  - **Modo básico** (sem sessão): até 5 ganhos, 5 despesas variáveis e 5 despesas
    fixas (limite total, não por mês). Ao tentar ultrapassar, aparece um aviso com
    "Entrar com Google". Editar e apagar continua a funcionar.
  - **Configurações bloqueadas no modo básico**: os interruptores, os campos de
    taxas, metas, turnos e plataformas ficam cinzentos e não respondem. Um banner
    explica porquê. Nome, foto, tema e modo escuro continuam disponíveis.
  - **Apagar tudo** também sai da conta Google neste aparelho e volta a mostrar as
    boas-vindas. Não altera a assinatura: a reposição escreve só o documento de
    dados do utilizador (`users/{uid}`), e a assinatura fica noutra coleção
    (`customers/{uid}`), que não é tocada.
  - **Decisões a confirmar**: (1) utilizadores existentes que já têm mais de 5
    registos mantêm tudo, mas não podem criar registos novos sem login; (2) o
    Apps Script (sincronização sem login) também conta como modo básico; (3) se um
    utilizador criou dados em modo básico e depois entra com uma conta que já tem
    dados na nuvem, prevalecem os da nuvem, como na regra de primeiro acesso.

- **Versão 0.4.30 — primeiro acesso, modo básico e estatísticas**:
  - **Dados locais antes da conta**: ao entrar com uma conta que já tem dados na
    nuvem, se este aparelho tiver dados que nunca foram sincronizados com essa conta,
    a app pergunta: "Manter os deste aparelho (substitui a nuvem)", "Começar de novo com
    os da nuvem (apaga os deste aparelho)" ou "Decidir depois". Se o aparelho já
    tinha sincronizado com a conta, a fusão com a base continua a ser usada, sem
    perguntar. Se a nuvem estiver vazia, os dados do aparelho são enviados sem perguntar.
  - **Modo básico**: além do IVA e da comissão, que podem ser ativados para testar,
    todas as outras configurações ficam bloqueadas.
  - **Estatísticas no modo básico**: mostram só o resumo do mês (bruto TVDE, pessoal,
    combustível, outras despesas e fixas, lucro). Dia, semana e ano ficam escondidos,
    e não há gráficos, turnos, km nem metas.
  - Pendente: a sincronização via Google Drive (Apps Script). Ver a conversa: o script
    atual usa um único ficheiro para todos os utilizadores.

- **Versão 0.4.31 — tela de carregamento sem conta**: sem sessão Google, mas com um
  URL de sincronização guardado, a app esperava pela sincronização antes de esconder
  a tela de carregamento. A ligação podia demorar até 15 s por tentativa, e a tela
  ficava presa. Agora a sincronização corre em segundo plano, como já acontece com
  a conta Google.

- **Versão 0.4.32 — modo básico sem planilha**: sem sessão Google, a app não lê nem
  envia para a planilha (Apps Script). Antes, com um URL guardado, a primeira
  sincronização perguntava se queria os dados da nuvem, mesmo sem conta. Agora
  `_doCloudSync()` e `pushToCloud()` saem logo quando não há sessão, e
  `scheduleCloudPush()` também. A planilha continua a funcionar para quem tem login.

## Versões 0.4.33 a 0.4.37 — Planilha, cópia diária, alerta de atualização e suporte

- **Versão 0.4.33 — endereço da planilha na conta e apagado com tudo**:
  - O endereço da planilha de sincronização passa a ser guardado em `profile.syncUrl`,
    no perfil da conta Google. Assim, segue a conta entre aparelhos. Sem sessão, não
    é guardado nem usado. Um endereço que já existisse neste aparelho passa para a
    conta no primeiro login.
  - **Apagar tudo** apaga também o endereço da planilha, neste aparelho e na conta.
    Redefinir só as configurações e apagar só os dados continuam a manter o endereço.
  - Nota: o endereço é uma credencial da planilha. Fica no documento da conta, que as
    regras do Firestore só deixam ler e escrever ao próprio utilizador.

- **Versão 0.4.34 — cópia diária na planilha**: com sessão e endereço da planilha
  configurado, ao abrir a app é enviada uma cópia do estado atual para a planilha,
  no máximo uma vez por dia (`localSettings.lastSheetBackup`, device-local). É só
  um envio: não lê nem funde dados da planilha. A cópia respeita a encriptação, se
  estiver ativa. Não corre enquanto houver uma decisão pendente sobre os dados. O
  resultado fica no log de sincronização (Modo Dev).
  - Limitação: a cópia só acontece quando a app é aberta. Se a app não for aberta
    num dia, a cópia desse dia não acontece.

- **Versão 0.4.35 — alerta de atualização sem sessão**: sem sessão Google, a app
  deixa de recarregar sozinha quando há uma versão nova. Mostra uma folha com a
  versão nova e três opções: "Atualizar agora", "Entrar com Google" (para receber
  as atualizações automaticamente) e "Mais tarde". Com sessão, a atualização
  automática continua igual. O alerta só aparece depois das boas-vindas.

- **Versão 0.4.36 — botão de suporte**: no fim de Configurações, abaixo da zona de
  perigo, o botão "✉️ Suporte" abre o email com o endereço de suporte já preenchido.
  O assunto e o corpo incluem o email e o ID da conta (ou "sem conta Google ligada"),
  e a versão da app. O botão funciona também em modo básico.

- **Versão 0.4.37 — suporte com log em ficheiro**: o texto do email passa a dizer
  "Mensagem:" (sem a palavra "problema"). O log das últimas 24 horas segue como
  ficheiro .txt (`corridaplus-log-AAAA-MM-DD-HHMM.txt`).
  - Se o aparelho permitir partilhar ficheiros (em geral, telemóvel), abre a folha de
    partilha com o ficheiro anexado e o texto do email. Esta folha não preenche o
    destinatário; o endereço aparece no texto.
  - Caso contrário, o ficheiro é transferido e o email abre com o endereço preenchido,
    pedindo para anexar o ficheiro. Um email não consegue anexar ficheiros por si.

## Versões 0.4.38 a 0.49 — Efeito transparente (liquid glass), gota e animações

- **Versão 0.4.38 — seleção da cor do tema visível no modo claro**: a cor escolhida
  tinha a borda na cor de texto, que no modo claro é escura e se confundia com a
  própria cor verde. Agora a seleção é um anel com um espaço claro à volta, visível
  em qualquer tema.

- **Versão 0.4.39 — liquid glass (iPhone), opção no Modo Dev**: nova secção "🪟 Visual"
  no Modo Dev, com o interruptor "Liquid glass (iPhone)". Guardado só neste
  aparelho (`localSettings.liquidGlass`).
  - Diagnóstico antes da alteração: o degradê da barra (`.tabbar`) tapava o que passa
    por trás; o interior da barra era quase opaco (branco a 90%, escuro a 97%); o
    fundo era liso, sem cor para refratar; cartões e folhas eram opacos.
  - Com o modo ativo: uma camada de cor fixa por trás de tudo (`body.glass::before`),
    barra com desfoque e saturação, cartões e folhas translúcidos, e o degradê da barra
    removido. Sem o modo, nada muda.
  - Limitação: é uma aproximação web. O material nativo "Liquid Glass" da Apple não
    está disponível num navegador, por isso o resultado pode diferir do iOS.

- **Versão 0.4.40 — efeito transparente em Configurações e nos controlos**:
  - O interruptor "Efeito transparente (liquid glass)" sai do Modo Dev e passa para
    Configurações → Aparência. Continua a ser uma preferência deste aparelho.
  - Animação das abas: no modo transparente, a troca de abas só anima a opacidade.
    O deslocamento vertical anterior fazia o vidro tremer.
  - Com o modo ativo, também ficam em vidro: a aba selecionada, os seletores
    Dia · Semana · Mês · Ano (com a opção ativa em vidro), as setas de navegação e
    os interruptores.

- **Versão 0.4.41 — opção ativa do seletor em verde**: com o efeito transparente, a
  opção selecionada de Dia · Semana · Mês · Ano voltou a ter o fundo da cor do tema
  (`--vault`), com texto branco, para indicar claramente que está ativa.

- **Versão 0.4.42 — separador e imagem de fundo**:
  - Um separador visível (linha) divide o bloco Aparência do bloco Configurações.
  - Em Configurações → Aparência, "Imagem de fundo": escolher uma imagem (reduzida para
    no máximo 1080 px, sem cortar) e remover. Fica guardada só neste aparelho
    (`localSettings.bgImage`) e aparece atrás de toda a app, com um véu para manter a
    legibilidade (claro ou escuro, conforme o modo).
  - Redefinir as configurações também remove a imagem de fundo.

- **Versão 0.4.43 — interruptor ligado com a cor do tema nos dois modos**: o interruptor
  ligado usa agora `--switch-on`, definida ao escolher a cor do tema. No modo
  transparente escuro, a regra do interruptor desligado sobrepunha o estado ligado,
  por isso ficava translúcido. Agora a regra só se aplica aos interruptores desligados.

- **Versão 0.4.44 — interruptor desligado visível sem o efeito transparente**: o trilho
  desligado usava uma cor muito translúcida (18%), que no modo escuro quase
  desaparecia. Agora é mais opaco no modo claro (24%) e tem uma cor própria no modo
  escuro normal. As regras do efeito transparente continuam a aplicar-se só com o
  efeito ativo.

- **Versão 0.4.45 — gota e gestos de navegação (efeito transparente)**:
  - **Gota da aba selecionada**: um indicador de vidro desliza até à aba nova com uma
    animação de mola (`#tabPill`, posicionado por `positionTabPill()`). A aba
    selecionada deixa de ter fundo próprio, porque quem o desenha é a gota.
  - **Arrastar sobre a barra**: ao arrastar, a gota segue o dedo, limitada às abas
    visíveis. Ao soltar, muda para a aba mais próxima. Um toque simples continua a
    funcionar como antes (`setupTabDrag()`).
  - **Deslizar sobre o conteúdo**: um gesto horizontal com pelo menos 70 px, mais
    horizontal do que vertical e feito em menos de 0,7 s muda para a aba seguinte
    ou anterior (`setupSwipeNav()`). Não atua sobre campos, botões, seletores,
    interruptores, gráficos nem com uma folha aberta.
  - Tudo só existe com o efeito transparente ativo. Sem ele, a barra funciona como antes.
  - Limitação: não testado num iPhone. Os gestos usam eventos de toque e de ponteiro,
    que o Safari suporta, mas a sensação de fluidez só se confirma no aparelho.

- **Versão 0.46 — arrastar a gota no celular**: no telemóvel, o navegador tomava o
  movimento horizontal como deslizar da página e cancelava o arrasto. Ao cancelar, a
  app usava coordenadas inválidas e voltava sempre à primeira aba. Agora a barra
  fica com o toque para ela (`touch-action: none`, só com o efeito transparente),
  a navegação usa a última posição real do dedo, e um cancelamento só devolve a gota
  à aba atual, sem mudar de aba.

- **Versão 0.47 — gota de vidro em todos os seletores (efeito transparente)**: a gota da
  barra de abas passa a existir em todos os seletores `.seg` (Dia · Semana · Mês · Ano,
  tema, frequência de despesas fixas, turnos). A opção ativa fica verde, com a cor do
  tema. Ao escolher, a gota chega achatada e recupera com mola. Ao arrastar, segue o
  dedo, estica com a velocidade e achata contra as paredes.



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

- **Versão 0.48 — deformação da gota também na vertical**: ao deslizar, estica até 20%
  no sentido do movimento e comprime até 18% na altura. Ao chegar, começa a 116% na
  largura e 82% na altura, e recupera com mola.

- **Versão 0.49 — animações também no modo normal**: a gota com deslize, deformação e
  mola funciona nos dois modos. No modo normal é sólida, com a cor do tema, e sem
  desfoque. O deslize sobre o conteúdo também muda de aba.

## Versões 0.50 a 0.56 — Mudanças suaves, imagem de fundo e legibilidade

- **Versão 0.50 — mudanças suaves**: trocar a cor do tema, o modo claro/escuro ou o
  efeito transparente faz as cores passar gradualmente (cerca de meio segundo). A
  camada de vidro aparece ou desaparece com opacidade. A imagem de fundo personalizada
  continua a mudar de uma vez.

- **Versão 0.51 — seletor de cor do tema com animação**: escolher uma cor já não
  redesenha a grade. Só a seleção muda nos círculos existentes, com mola e um pequeno
  encolhimento ao tocar.

- **Versão 0.52 — imagem de fundo com fade**: a camada da imagem (`#customBg`) deixou
  de estar oculta com `display:none`, o que impedia qualquer transição. Agora:
  - Ao ativar ou desativar o efeito transparente, a imagem entra de novo com fade.
  - Ao trocar de imagem ou remover, a atual sai (fade), a nova entra.
  - Ao mudar o modo claro/escuro, a imagem troca o véu com a mesma transição.

- **Versão 0.53 — imagem de fundo sem esbranquiçar no modo claro**: o véu no modo claro
  era creme a 55%, o que lavava a imagem. Agora é um véu escuro leve (22%), que mantém
  as cores da imagem e continua legível. O modo escuro não muda (véu escuro a 62%).

- **Versão 0.54 — legibilidade com imagem de fundo no modo claro**: ao escolher a imagem,
  a app calcula o tom médio (luminosidade, já com o véu). Se for escura, no modo claro
  o texto que está diretamente sobre o fundo (títulos, datas dos seletores, mensagens
  vazias, linhas de despesas, botão "Adicionar") fica claro. Os cartões brancos e os
  campos mantêm o texto escuro. Imagens antigas são analisadas na primeira abertura.
  O modo escuro não muda.
  - Limitação: a média pode não bater em imagens com partes muito escuras e muito
    claras ao mesmo tempo. Nesse caso, o texto pode continuar pouco legível numa das zonas.

- **Versão 0.55 — ligar/desligar o efeito transparente sem piscar**: a função do
  interruptor redesenhava todo o ecrã de Configurações (`renderSettings()`), e a imagem
  de fundo repetia o fade. Agora só o interruptor é atualizado, e a imagem não é
  reanimada. Assim, o efeito muda como a troca de tema, que já não piscava.

- **Versão 0.56 — legibilidade com imagem de fundo e fade ao ligar o efeito**:
  - **Ligar/desligar o efeito transparente**: a interface e o menu inferior saem com fade,
    o efeito troca, e voltam com fade (≈0,2 s), em vez de piscar.
  - **Balão Detalhamento (Lucro)**: usava uma cor de fundo inexistente (`var(--card)`),
    por isso ficava transparente sobre a imagem. Agora é um cartão normal.
  - **Textos com cor fixa**: no modo claro com imagem, os textos com cor muted/faint
    escritos no HTML (ex.: descrição de Despesas fixas, rótulos de ano) ficam claros
    sobre o fundo. Dentro de cartões brancos voltam à cor normal.
  - **Botão "Adicionar" no modo escuro com imagem**: fundo escuro translúcido e texto
    claro. Sem imagem, nada muda.

## Versões 0.57 a 0.59 — Intensidade do efeito e da imagem de fundo

- **Versão 0.57 — sem efeito de subida ao desligar o efeito transparente**: a animação
  das telas no modo normal deslizava 6 px de baixo para cima. Ao desligar o efeito, o
  nome da animação mudava e ela recomeçava, parecendo que a página subia. Agora a
  animação das telas é só de opacidade, nos dois modos. A troca de abas no modo normal
  deixa de ter o pequeno deslize.

- **Versão 0.58 — intensidade do efeito transparente e balões tintados**: com o efeito
  ligado, Configurações → Aparência mostra uma barra de **intensidade** (0 a 100, padrão
  50). Os valores intermédios são possíveis. O rótulo muda entre "Mais transparente",
  "Padrão" e "Mais opaco". Escala a transparência e o desfoque das barras, cartões,
  folhas e botões (`--gl-k` e `--gl-b`). Com o padrão (50), o aspeto é o atual.
  - **Balões tintados** (ligado por omissão): a gota e os seletores usam a cor do tema.
    Desligado: neutros e sem cor.
  - Guardado só neste aparelho (`localSettings.glassLevel` e `localSettings.glassTint`).

- **Versão 0.59 — ajustes da imagem de fundo**: só aparecem quando há uma imagem.
  - **Véu sobre a imagem** (0 a 100%): 0% é o véu leve atual; 100% é o véu totalmente
    opaco, que tapa a imagem.
  - **Desfoque com granulação** (0 a 100%): aplica desfoque (até 20 px) e uma camada de
    ruído com modo de mistura "sobreposição", que cresce com o desfoque.
  - Os dois ajustes atualizam ao vivo, sem fade. Só trocar de imagem tem fade.
  - Guardado só neste aparelho (`localSettings.bgVeil` e `localSettings.bgBlur`).
  - Limitação: o desfoque em ecrãs de telemóvel pode pesar. Se a app ficar lenta com
    o desfoque no máximo, reduzo o valor máximo.

## Versões 0.60 a 0.63 — Navegação entre abas

- **Versão 0.60 — troca de abas com deslize, sem fade**: desde a 0.56 e anteriores, as
  telas tinham uma animação de opacidade (0 a 100%), e os cartões pareciam transparentes
  a ficar opacos, como se a página estivesse a carregar. Agora:
  - A nova aba entra com um deslize de 28 px, do lado de onde vem (direita ao avançar,
    esquerda ao voltar), com opacidade total desde o início.
  - A ordem é a das abas da barra (`SCREEN_ORDER`). Trocar pelos gestos ou pela barra
    usa a mesma animação.
  - Limitação: a aba que sai não faz deslize de saída; desaparece de imediato.

- **Versão 0.61 — deslizar na página segue o dedo**:
  - **Página** (só telemóvel): ao deslizar na horizontal, a aba atual acompanha o dedo e
    a vizinha aparece do lado. Ao soltar, muda se o deslize passar 25% da largura ou for
    rápido; senão volta. A vizinha é a da ordem da barra.
  - Não atua sobre campos, botões, seletores, interruptores nem gráficos, nem com uma
    folha aberta. Em ecrãs largos (computador), o deslize não está ativo.
  - Limitação: não testado num iPhone. A sensação de seguir o dedo depende do aparelho.

- **Versão 0.62 — barra de abas volta a arrastar a gota**: a alteração da 0.61 que impedia o
  arrasto na barra estava errada. Ao deslizar sobre a barra, a gota volta a seguir o dedo,
  e ao soltar muda para a aba mais próxima, como antes.

- **Versão 0.63 — removido o deslize da página**: o deslize na horizontal sobre o conteúdo
  que mudava de aba (das versões 0.61 e 0.62) foi removido. Mantém-se a animação de deslize
  quando se toca numa aba (a nova entra pelo lado de onde vem). Continua a funcionar o arrasto
  da gota na barra de abas.

## Versões 0.64 a 0.67 — Barra de abas fixa e cantos concêntricos

- **Versão 0.64 — barra e fundo estáveis ao trocar de aba (diagnóstico, sem iPhone)**:
  ao trocar de aba, a app chamava sempre `window.scrollTo(0,0)`. No iOS, isso faz a barra de
  endereço recolher ou expandir, e as camadas fixas (barra, fundo e camada de vidro) podem
  saltar ou parecer redimensionadas. Agora só volta ao topo se a página estiver rolada, e
  as camadas fixas ficam em composição própria (`translateZ(0)`), para não serem repintadas
  na troca de aba. Não foi possível reproduzir: confirmar no iPhone.

- **Versão 0.65 — cantos aninhados concêntricos**: aplicada a regra "raio interno = raio externo
  − espaçamento". Seletores (Dia·Semana·Mês·Ano, tema, frequência, turnos): contentor 12px,
  espaçamento 3px, opções e gota (valores finais na 0.66). Barra de abas: contentor 20px. Os círculos do seletor de cor
  e os interruptores não precisam de ajuste.

- **Versão 0.66 — cantos concêntricos com a borda contada**: os valores passam a contar a
  borda, além do espaçamento. Seletores: 12 − (1,5 + 3) = 7,5px nas opções e na gota. Barra
  de abas: 20 − (1 + 7) = 12px nas abas e na gota. Computador (barra lateral): 20 − (1 + 12) = 7px.

- **Versão 0.67 — barra de abas fixa (Android e iOS) e efeito transparente sem reiniciar**:
  - **Causa**: a página inteira rolava. Ao rolar, a barra de endereço do navegador recolhe ou
    expande, e a barra de abas, que é fixa, saltava de lugar. Acontecia em páginas mais compridas
    e no Android. A camada `translateZ` da 0.64 foi removida, porque não resolvia.
  - **Correção**: só a área das telas (`#app`) rola. O corpo da página fica fixo, e a barra de
    abas nunca se mexe. Ao trocar de aba, a área volta ao topo.
  - **Efeito transparente**: o fade ao ligar ou desligar termina sempre (`finally`), mesmo que
    algo falhe. Assim a interface não fica a meio fade e não precisa de reiniciar.

## Versões 0.68 a 0.70 — Ímã, correções e legibilidade das barras

- **Versão 0.68 — "ímã" na intensidade do efeito transparente**: a barra de intensidade encaixa
  em 0 (transparente mais), 50 (padrão) e 100 (opaco), com uma margem de 6 pontos. Perto
  desses valores, o cursor vai para a posição exata. Os valores intermédios continuam
  disponíveis, longe dos três pontos.

- **Versão 0.69 — correção: temas em tempo real**: a função `softChange()` tinha sido apagada
  por engano numa alteração anterior. Por isso, ao mudar a cor do tema, o modo claro/escuro
  ou o efeito transparente, a mudança falhava e a animação gradual não corria. Reposta.
  Confirmado que não falta mais nenhuma função de código.

- **Versão 0.70 — barras e textos legíveis; cantos concêntricos explícitos**:
  - **Barras de intensidade e do fundo** (véu, desfoque): polegar branco com contorno na cor
    do tema, trilho mais escuro e parte preenchida na cor do tema. Nos dois modos.
  - **Textos das barras** dentro de um cartão próprio (fundo branco no modo claro, escuro no
    modo escuro), para ficarem legíveis mesmo com imagem de fundo.
  - **Cantos**: seletores com raio interno = 12 − (1,5 + 3) = 7,5px, e barra de abas com 20 − 8
    = 12px, agora definidos por variáveis (`--seg-inner`, `--tab-inner`).
  - Limitação: os cartões menores dentro de outros cartões (ex.: estatísticas) têm espaçamento de
    14–16px num cartão de raio 16px. A regra daria quase um canto reto; por isso não foram
    alterados. Se quiseres a regra aplicada à risca, reduzo o espaçamento interno ou o raio externo.

## Versões 0.71 a 0.73 — Balões, imagem de fundo e cantos no vidro

- **Versão 0.71 — balões sempre tintados; interruptor da imagem de fundo**:
  - Removida a opção "Balões tintados". Os balões ficam sempre com a cor do tema.
  - Em Configurações → Aparência, a imagem de fundo tem um interruptor de ativar ou desativar.
    Desligada, a imagem fica guardada e volta a aparecer ao ligar. Por omissão, fica ativa
    quando existe imagem (`localSettings.bgEnabled`). O modo claro com texto claro sobre o
    fundo também só se aplica com a imagem ativa.

- **Versão 0.72 — rótulo "Padrão" alinhado ao ímã**: os rótulos Transparente / Padrão / Opaco
  estavam distribuídos com espaço entre eles, e o do meio ficava fora do ponto 50%. Agora o
  "Padrão" fica centrado exatamente na posição 50%, onde o cursor encaixa.

- **Versão 0.73 — cantos concêntricos também no modo transparente**: a regra estava definida,
  mas no modo transparente uma regra anterior fixava as gotas em 14px, mais do que o contentor
  (12px) permite. Por isso a gota ficava fora de alinhamento com a moldura. Agora as gotas e as
  opções dos seletores, e as abas da barra, têm prioridade e seguem a regra nos dois modos.

## Versões 0.74 a 0.76 — Seletores: animações e interruptor da imagem

- **Versão 0.74 — seletor de tema desliza ao clicar**: a mudança suave de cores (0.50) aplicava
  uma transição de cores a todos os elementos, incluindo a gota. Por isso, ao clicar numa opção
  de tema (Claro · Sistema · Escuro), a gota saltava em vez de deslizar. Agora a gota mantém a
  sua animação de deslize durante a mudança de tema. Arrastar já funcionava.

- **Versão 0.75 — intensidade sem preenchimento verde**: a barra de intensidade do efeito
  transparente deixa de pintar a parte à esquerda da bolinha. Fica só o trilho neutro e a
  bolinha. As barras do véu e do desfoque da imagem de fundo mantêm o preenchimento.

- **Versão 0.76 — interruptor da imagem de fundo com animação**: ao ativar ou desativar, a linha
  inteira era redesenhada, e o interruptor aparecia já no estado final. Agora o interruptor é criado
  uma vez e só muda de cor. Só o conteúdo abaixo (pré-visualização, botões e barras) é redesenhado.

## Versões 0.77 a 0.80 — Legibilidade e ajustes finais

- **Versão 0.77 — barra de intensidade aparece logo ao ativar o efeito transparente**: a barra
  só era desenhada ao renderizar as Configurações, por isso só aparecia depois de mudar de tela e
  voltar. Agora o ligar e desligar atualiza a barra na hora.

- **Versão 0.78 — "Conectado com Google" legível no modo escuro**: o texto usava o verde escuro
  do modo claro, quase invisível sobre o fundo escuro. Agora usa a cor de texto do tema no modo
  escuro (`--vault-text`), e continua igual no modo claro. Vale para o modo normal e o transparente.

- **Versão 0.79 — distintivos de plataforma padronizados**: Uber, Bolt, Particular, Outros e as
  plataformas extra usavam formas, letras (a Bolt com emoji) e tamanhos diferentes. Agora um único
  distintivo: quadrado arredondado de 8px, letra centrada, cor fixa por plataforma (U preto, B verde,
  P azul, O castanho, extras roxo). Dois tamanhos apenas: 16px dentro de títulos e 28px nas listas
  (ganhos, configurações).

- **Versão 0.80 — títulos legíveis no modo escuro**: o "Corrida+" da barra lateral (computador) e os
  títulos das telas ("Você", "Ganhos", etc.) usavam o verde escuro do tema, pouco contrastado no modo
  escuro. Agora usam a cor de texto do tema nesse modo. O modo claro não muda.

## Versão 0.81 — Modo escuro gradual, nome sem fechar o teclado e barras do sistema

- **Seletor Claro · Sistema · Escuro com transição gradual.** A troca não reconstrói
  mais as Configurações: só atualiza a opção ativa. Antes, o redesenho do ecrã
  apagava as transições das cores dos cartões.
- **Nome do perfil guardado só ao confirmar.** Passou de `oninput` para `onchange`
  (ao sair do campo ou com Enter). Antes, cada letra disparava uma gravação, que
  por sua vez reconstruía o ecrã e fechava o teclado.
- **Cor da barra do sistema (Android).** A barra de cima e a de baixo seguem a cor do
  tema. Com o efeito transparente ativo, usam a cor de base do vidro (creme no modo
  claro, quase preto no escuro). O elemento `meta theme-color` é recriado a cada
  mudança, porque o Chrome nem sempre atualiza só o atributo.
  - Limitação: uma página web não consegue tornar transparente a barra do sistema do
    Android. A aproximação é a cor de fundo do vidro. Pode ser preciso reabrir a app
    instalada para a cor nova aparecer.

## Versão 0.82 — Transição de cor da gota nos seletores

- **Gota do seletor com cor gradual ao mudar o tema.** A regra `html.soft-change .glass-pill`
  definia a lista de propriedades animadas com `!important` e não incluía a cor de fundo.
  Assim, a gota (a opção ativa do seletor e da barra) trocava de cor de repente, enquanto
  o resto da app fazia a transição. Agora `background-color`, `border-color` e `box-shadow`
  fazem parte da transição.

## Versão 0.83 — Cor da barra de navegação do Android acompanha o tema

- **Causa:** a barra de baixo (e de cima) da app instalada ficava verde porque o Android
  usa o `theme_color` do manifest (`#1F3A2E`), e não só a tag `meta theme-color`.
- **Correção:** ao mudar o tema, o modo claro/escuro ou o efeito transparente, a app
  atualiza também o `theme_color` e o `background_color` do manifest, com a mesma cor.
- **Limitação:** o Chrome relê o manifest ao abrir a app. A nova cor pode só aparecer ao
  fechar e reabrir a app instalada (ou reinstalá-la).

## Versão 0.84 — Assinatura expirada e aviso de validade

- **Causa:** o app só lia o `status` da assinatura e nunca comparava `current_period_end`
  com a data de hoje. Um documento com `status: "active"` e data já passada continuava a
  aparecer como ativo, e o Firestore não enviava nenhuma alteração com a passagem do tempo.
- **Correção:** o cartão "Assinatura" compara a data de fim com a data atual:
  - depois da data: "⛔ Assinatura expirada", com a data em que o período terminou;
  - nos últimos 7 dias: aviso com os dias que faltam;
  - o cartão é reavaliado a cada minuto, sem depender de alterações no Firestore.
- **Limitação:** este cartão apenas informa. O bloqueio real de acesso fica para a Fase 3
  (regras do Firestore e extensão Stripe), e ainda não existe.

## Versão 0.85 — Modo só leitura quando a assinatura expira

- **Regra:** com assinatura expirada (data de fim já passada), a conta fica só para leitura:
  vê os dados, mas não regista, edita nem apaga. Sem tolerância de dias.
- **App:** um aviso fixo no topo de todos os ecrãs indica a data de expiração e tem o botão
  "Renovar assinatura" (por agora, leva a Configurações → Conta; o checkout do Stripe ainda
  não está ligado). Todas as ações de escrita passam por `readOnlyGuard()`, que mostra um aviso
  e cancela a ação.
- **Servidor (o que realmente bloqueia):** a Cloud Function `updateAccess`
  (`functions/index.js`) recalcula, sempre que uma assinatura muda, até quando o acesso é válido
  e grava `access/{uid}.until`. As regras (`firestore.rules`) só permitem escrever em
  `users/{uid}` quando esse campo está no futuro.
- **Assunção:** sem assinatura nenhuma (ainda sem Stripe), a app continua livre. Quando o Stripe
  estiver ligado, isto deve ser revisto: as regras negam escrita sem `access/{uid}`.
- **Publicação:** primeiro a função (`firebase deploy --only functions`), depois as regras
  (`firebase deploy --only firestore:rules`). A função só corre quando uma assinatura muda, por isso,
  para a conta de teste, basta editar um campo do documento em `customers/{uid}/subscriptions`
  para criar `access/{uid}`.

## Versão 0.86 — Sub-ecrã de Aparência

- **Motivo:** a secção Aparência dentro de Você tinha muitos controlos (modo, cor, efeito
  transparente, imagem de fundo), e as pessoas achavam que Configurações tinha só isso.
- **Mudança:** Você mostra agora um botão "🎨 Aparência ›". Abre o ecrã `screen-aparencia`, que
  não está na barra de abas e tem uma seta para voltar a Você. O conteúdo é o mesmo, só mudou de sítio.
- **Efeito colateral corrigido:** no modo básico, as regras de bloqueio das Configurações
  também desativavam o efeito transparente e a imagem de fundo. Agora estão no ecrã próprio,
  fora da lista bloqueada. (Só o aparência de cor e modo escuro já funcionava no modo básico.)

## Versão 0.87 — Toque longo na barra de abas

- **Sintoma:** manter o dedo em "Você" na barra de abas mostrava a barra de carregamento
  no topo e a app recarregava depois de cerca de 3 segundos.
- **Causa provável (não confirmada):** o gesto de toque longo chegava ao navegador, que
  o interpretava como menu ou recarregamento. Nenhuma função da app recarrega ao tocar nas abas.
- **Correção:** a barra de abas deixa de mostrar o menu de toque longo e de selecionar texto;
  o gesto de puxar para atualizar fica desativado (`overscroll-behavior: none`).
- **A confirmar:** se o recarregamento continuar, é preciso saber se é a app instalada ou o
  navegador, porque o comportamento pode depender do navegador do aparelho.

## Versão 0.88 — Manter "Você" para procurar atualização

- **Gesto:** manter o dedo em "Você" na barra de abas (cerca de 0,7 s) procura uma atualização.
  Se houver uma versão nova, a app atualiza-se sozinha, sem perguntar. Se já estiver na versão
  mais recente, mostra um aviso com a versão atual. Um toque normal continua a abrir Você.
- **Como funciona:** lê a versão publicada (`APP_VERSION` na página) e, se for diferente,
  guarda-a em `pendingUpdateNotice` e recarrega a página. Na abertura seguinte aparece o aviso
  "Atualizado para vX.X.X" no ecrã de carregamento, como já acontecia.
- **Substitui o comportamento da 0.87:** o toque longo deixou de ser só bloqueado.

## Versão 0.89 — Gesto "toque e depois manter" em Você

- **Gesto:** um toque em "Você" e, menos de 1,5 s depois, manter o dedo cerca de 0,7 s
  procura uma atualização (ver 0.88). Manter o dedo sem um toque antes não faz nada, para
  evitar atualizações por engano.
- Um toque normal continua a abrir Você.

## Versão 0.90 — Transição do sub-ecrã de Aparência

- O sub-ecrã Aparência entrava de repente. Agora entra com o mesmo deslize das abas
  (pela direita) e sai com a seta, que o faz deslizar para a esquerda, como ao mudar de aba.

## Versão 0.91 — Interruptor do gesto em Você (Modo Dev)

- Modo Dev → Atualização → "Atualizar ao manter Você". Ligado por omissão.
- Guardado só neste aparelho (`localSettings.holdUpdateEnabled`), porque o Modo Dev também é deste aparelho.
- Desligado, o toque longo em Você não procura atualizações; o toque normal continua a funcionar.

## Versão 0.92 — Cor de destaque da aba acompanha a gota

- **Antes:** a aba ativa ficava inteira com a cor de destaque, e a gota só mudava de posição.
  Ao deslizar a gota, a cor ficava na aba antiga até a gota sair.
- **Agora:** a cor de destaque só aparece dentro da gota. Uma camada `.tab-ov` (cópia das abas)
  fica por cima da barra e é recortada pela forma da gota (`clip-path`). Se a gota estiver a
  meio de um ícone, só essa metade fica colorida, como no iOS. O recorte acompanha a gota no arrasto
  e na animação.
- **Nota:** o ícone da aba ativa deixou de ter a ampliação de 8%, para as duas camadas ficarem iguais.
- **Fora desta versão:** os seletores Dia · Semana · Mês continuam a mudar de cor de uma vez.

## Versão 0.93 — Correção da cor de destaque no modo escuro

- A camada de cor da 0.92 ficava cinzenta no modo escuro: a regra `body.dark .tab` tinha mais
  peso do que a herança da cópia. Agora a cópia força a herança da cor (`!important`).
- Testado num navegador headless nos modos claro, efeito transparente e escuro.

## Versão 0.94 — Sem texto duplicado ao parar a gota

- **Sintoma:** quando a gota parava, a cópia colorida ficava por cima do texto cinzento e as duas
  palavras sobrepunham-se (efeito de texto em negrito ou com sombra).
- **Correção:** duas camadas que nunca se sobrepõem. `.tab-base` (cinzento) é recortado para ficar
  fora da gota (`path()` com regra evenodd). `.tab-ov` (cor de destaque) fica só dentro da gota.
  As abas reais ficam transparentes e só recebem os toques.
- Testado num navegador headless nos modos claro, efeito transparente e escuro.

## Versão 0.95 — Turnos: Dia e Noite, e pergunta do período no ganho

- **Turnos padrão:** agora são só **Dia** e **Noite**. Manhã, Tarde e Madrugada deixam de ser padrão.
  Continua a ser possível adicionar outros turnos em Configurações → Turno de trabalho.
- **Migração:** quem já tinha Manhã, Tarde ou Madrugada na lista vê-os arquivados (aparecem em
  "Removidos" e podem ser reativados). Os ganhos antigos mantêm o nome que tinham.
- **Novo ganho:** a pergunta do seletor passou a ser "Qual período trabalhou mais?".

## Versão 0.96 — Turnos antigos saem das Configurações

- Manhã, Tarde e Madrugada deixam de aparecer em Configurações, inclusive em "Removidos".
  A migração remove-os da lista guardada.
- Os ganhos antigos com esses turnos continuam a mostrar o nome certo no gráfico "Ganhos por
  turno" e na exportação, através do mapa `TURNOS_ANTIGOS` (só para leitura).

## Versão 0.97 — Período do lançamento obrigatório e dias no gráfico por turno

- Os lançamentos são feitos uma vez por dia, no fim do turno. Por isso, a escolha do período
  deixa de ter "Não dizer": o primeiro período ativo vem selecionado por omissão.
  Assim, todos os lançamentos entram no gráfico "Ganhos por turno".
- O gráfico indica quantos dias contou o período que mais rendeu ("Melhor turno: Dia · 12 dias").

## Versão 0.98 — "Não dizer" volta ao seletor de período

- Com os turnos ativados, o seletor "Qual período trabalhou mais?" volta a ter a opção
  "Não dizer", selecionada por omissão. A alteração da 0.97 que a removia foi revertida.
- Mantém-se o contador de dias no gráfico "Ganhos por turno".

## Versão 0.99 — Dias trabalhados por período nas Estatísticas

- O gráfico "Ganhos por turno" mostra, por período, quantos dias foram lançados e as datas
  (dd/mm). Ex.: "☀️ Dia · 5 dias" e por baixo "03/10 · 04/10 · …".
- Conta cada data uma vez por período, dentro do intervalo que está em ecrã (dia, semana, mês ou ano).

## Versão 0.100 — Estatísticas mais completas por período

- **Por período (Dia e Noite):** a lista do gráfico "Ganhos por turno" mostra o total ganho e a
  média por dia de cada período.
- **Melhores dias:** novo cartão com os 5 dias de maior ganho bruto (soma de todas as plataformas
  nesse dia), com o dia da semana, a data e o período. Aparece em Estatísticas → Mês e → Ano.

## Versão 0.101 — Análise completa nas Estatísticas

Novo bloco, no fim das Estatísticas de Mês e de Ano:

- **Atividade:** dias trabalhados (e de quantos do mês), média por dia, melhor e pior dia.
- **Por dia da semana:** média de ganho em cada dia da semana, com o melhor destacado.
- **Ganhos acumulados no mês (só Mês):** linha que mostra como o total foi crescendo ao longo do mês.
- **Comparado com o mês anterior (só Mês):** bruto e lucro, com a variação em percentagem.

Os cálculos usam os mesmos valores do resto das estatísticas (IVA, comissão, despesas fixas e variáveis).

## Versão 0.102 — Barra lateral alinhada no computador

- No ecrã largo, a barra lateral ficava desalinhada: as camadas de cor subiam cerca de 25 px.
  A causa era o título "Corrida+" existir só na barra original. Agora as camadas têm o mesmo
  título, escondido, e as posições ficam iguais às das abas.

## Versão 0.103 — Barra de rolagem discreta no computador

- A barra de rolagem padrão do navegador aparecia branca e larga no lado direito. Agora é fina,
  com fundo transparente e a cor do tema (claro ou escuro). A rolagem com o rato continua igual.

## Versão 0.104 — Opção "Nas semanas" nas despesas fixas mensais

- Com a opção padrão ("não aparecer nas semanas"), a despesa não mostra nada extra.
- Com "dividir pelas semanas" ou "tudo na 1ª semana", aparece uma linha legível com o resumo e
  o botão "alterar".
- "alterar" abre uma folha com as três opções. Também permite voltar à opção padrão.
- O seletor branco da lista, que ficava ilegível no modo escuro, foi removido.

## Versão 0.105 — Barra de rolagem discreta também nas folhas

- As folhas (Novo ganho, despesas, etc.) e o registo de sincronização tinham a barra de rolagem
  padrão do navegador. Agora usam o mesmo estilo fino e transparente da área principal.

## Versão 0.106 — Ícone da tela de carregamento com a cor do tema

- O ícone da tela de carregamento e o da tela de login tinham cores fixas (verde e dourado).
  Agora usam as variáveis do tema: o bloco com `--vault-2` e os detalhes com `--brass-light`.
- Testado com o tema roxo: o ícone acompanha a cor escolhida.
- Limitação: o ícone instalado no ecrã inicial (manifest e favicon) continua fixo, porque o
  sistema só o lê na instalação.

## Versão 0.107 — Ícone do app personalizável (dentro da app)

- Aparência → "Ícone do app": escolhe a cor (igual ao tema ou uma das cores do tema) e liga
  ou desliga o efeito transparente (bloco translúcido com contorno claro).
- Aplica-se à tela de carregamento e ao login. Guardado só neste aparelho (`localSettings.iconColor`,
  `localSettings.iconGlass`).
- **Limitação:** o ícone instalado no ecrã inicial (manifest e favicon) só muda com nova instalação
  ao usar o navegador. Numa versão em APK, o ícone pode seguir estas mesmas escolhas.

## Versão 0.108 — Pré-visualização do ícone

- Aparência → "Ícone do app": no topo da secção aparece o ícone em tamanho grande, sobre o mesmo
  fundo da tela de carregamento. Atualiza logo ao mudar a cor ou o efeito transparente.

## Versão 0.109 — Tela de carregamento com a cor certa desde o início

- **Causa:** a cor do tema e do ícone só eram lidas depois do carregamento, por isso a tela
  mostrava primeiro as cores por omissão e só depois mudava.
- **Correção:** um pequeno script no início da página lê as preferências guardadas neste aparelho
  (`localSettings`) e aplica o tema e o ícone antes da primeira pintura.
- A tela de carregamento fica visível pelo menos 1,2 s, para o ícone poder ser visto.

## Versão 0.110 — Ícone da tela de carregamento começa cinzento

- O ícone da tela de carregamento começa em cinzento (filtro `grayscale`), e passa para a cor
  escolhida com transição de 0,9 s quando a sincronização termina.
- A app espera até 6 s pela sincronização. Se a rede estiver lenta, a cor aparece na mesma
  ao fim desse tempo. Sem conta (modo básico), a cor aparece logo após carregar os dados locais.
- A pré-visualização nas Aparências continua a mostrar o ícone a cores.

## Versão 0.111 — Ícone de arranque (manifest) igual ao da app

- A primeira tela que aparece ao abrir a app instalada é a tela de arranque do sistema. O Android
  cria-a a partir do manifest guardado na instalação, por isso não é a tela de carregamento da app.
- O ícone do manifest foi atualizado para o mesmo desenho da tela de carregamento (carro preenchido,
  cores padrão). Este ícone só muda depois de instalar a app de novo pelo navegador.
- Limitação: como no APK, a cor escolhida em Aparência não se aplica a esta tela de arranque. Para
  ela seguir o tema, é preciso gerar o ícone no momento da build do APK.

## Versão 0.112 — Mudança de cor do ícone gradual e padrão de seletores

- **Ícone:** ao mudar a cor ou o efeito transparente, a alteração é gradual (`softChange`), e a
  pré-visualização anima como o resto da app.
- **Seletor de cores:** só muda a classe `.selected` (com a mesma animação do seletor de tema). A
  grelha e a pré-visualização não são recriadas, porque isso apagava a transição.
- **Nota de padrão:** foi escrita no código, junto a `softChange`, para servir de regra em toda a app:
  mudanças visuais graduais, seletores que só trocam classes, e nada de `innerHTML`/`display:none`
  em elementos que precisam de animar.
- Testado: a cor da pré-visualização passa por um tom intermédio antes de chegar à cor final.

## Versão 0.113 — Fundo da tela de carregamento segue o tema

- O fundo da tela de carregamento tinha uma cor fixa no modo escuro do sistema, por isso ignorava o
  tema escolhido. Agora usa `--splash-bg`: a cor do tema no modo claro e uma versão mais escura da mesma
  cor no modo escuro (`html.splash-dark`).
- A marca `splash-dark` é aplicada antes da primeira pintura, a partir das preferências guardadas, e
  atualizada ao mudar o modo claro/escuro/sistema.
- A pré-visualização do ícone usa o mesmo fundo.

## Versão 0.114 — Diferença clara entre modo claro e escuro no fundo

- Com o tema verde por omissão, o modo escuro era só um verde ligeiramente mais escuro, e a troca
  quase não se via na pré-visualização. Agora o modo escuro mistura a cor do tema com quase preto
  (35% da cor, 65% de preto), por isso a diferença é clara.

## Versão 0.115 — Escolha do fundo da tela de carregamento

- Aparência → "Fundo da tela de carregamento":
  - **Igual ao tema** (predefinido): a cor do tema no modo claro e uma versão mais escura no modo escuro;
  - **Fundo do app**: creme no modo claro e escuro no modo escuro;
  - **Uma cor do tema**: fixa, independente do tema escolhido.
- O texto da tela de carregamento (título e mensagens) muda para ficar legível sobre o fundo
  escolhido (`--splash-fg`), e a pré-visualização do ícone usa o mesmo fundo.
- A regra de fundo fica numa única função (`computeSplash`), partilhada pelo arranque e pela app.

## Versão 0.116 — Ícone da aba do navegador acompanha as configurações

- O favicon (ícone da aba do navegador) era fixo. Agora é gerado com a cor do tema (ou a cor escolhida
  para o ícone) e com o efeito transparente, como o ícone da app.
- Os navegadores atualizam o favicon ao vivo, por isso a mudança aparece logo, sem reinstalar.
- O favicon é aplicado logo no arranque, a partir das preferências guardadas, e atualizado ao mudar
  o tema ou o ícone.

## Versão 0.117 — Rótulos legíveis no modo escuro

- Os rótulos (por exemplo "MODO ESCURO", "TAXA DE COMISSÃO", "EMOJI" e "NOME") usavam a cor escura de
  destaque do tema, quase invisível no modo escuro. Agora usam a cor clara do tema
  (`--vault-text`), como o texto "Conectado com Google".

## Versão 0.118 — Configurações organizadas em blocos

- A secção Configurações tinha todas as opções seguidas e misturadas. Agora cada assunto está num
  bloco próprio, separado por uma linha:
  - **Impostos:** IVA e comissão, juntos (são do mesmo tema);
  - **Turnos:** turno de trabalho e o editor de nomes e emojis;
  - **Quilómetros**, **Metas de lucro**, **Outras plataformas** e **Resumo de ontem**, cada um sozinho.
- Os identificadores e o comportamento das opções não mudaram.

## Versão 0.119 — Ordem das opções em Configurações

- Nova ordem: IVA e comissão (Impostos), metas de lucro, turno de trabalho, quilómetros, outras
  plataformas e resumo de ontem. Só muda a ordem dos blocos; o comportamento não mudou.

## Versão 0.120 — Correção: abas vazias (Lucro, Estatísticas, Dev)

- Ao reorganizar Configurações (0.118/0.119), faltava um `</div>` no último bloco. Assim, o ecrã de
  Configurações não fechava e os ecrãs seguintes (Estatísticas, Lucro, Dev) ficavam dentro dele.
  Como Configurações está escondido, esses ecrãs ficavam vazios.
- Corrigido o fecho do bloco; todos os ecrãs passam a ser filhos diretos da área principal.

## Versão 0.121 — Balão de sincronização alinhado com o título

- Em Você, o balão "sincronizado" ficava acima do nome da aba, porque a coluna da direita (balão + versão)
  era centrada no conjunto. Agora o balão fica centrado no título, e o número da versão aparece
  por baixo dele, fora da altura do cabeçalho.

## Versão 0.122 — Primeiros ajustes de Gestalt (figura/fundo, rótulos, separação)

- **Figura e fundo:** no efeito transparente, os cartões passam a ter mais opacidade (78%, 80% no
  modo escuro) e um contorno claro, para se destacarem da imagem de fundo.
- **Rótulos:** uma única variável (`--label-color`) define a cor dos rótulos em cada modo, em vez de
  cores diferentes espalhadas pelo código.
- **Você:** mais espaço antes de "Configurações" e entre o botão "Aparência" e a secção seguinte.
- **Ainda por fazer:** destaque único do número principal de cada ecrã (ponto 4 da análise).

## Versão 0.123 — Número principal em destaque e zonas na Você

- **Destaque único por ecrã (ponto 4):** cada ecrã tem agora um número principal no topo, com o mesmo
  estilo do cartão de lucro:
  - Estatísticas (mês): lucro do mês (verde, ou terracota se negativo);
  - Despesas: total do mês, fixas + variáveis (terracota);
  - Ganhos e Lucro: já tinham o destaque do total bruto e do lucro líquido.
- **Você (ponto 2):** a secção de aparência tem agora um cabeçalho "Preferências", separado de "Conta"
  e de "Configurações".
- Função reutilizável `heroValorHTML(rótulo, valor, subtítulo, cor)`.

## Versão 0.124 e 0.125 — Revisão de todos os ecrãs (Gestalt)

Revisão feita nos sete ecrãs, nos modos claro e escuro, com dados de exemplo.

- **Semelhança:** o Total bruto de Ganhos passa a usar o mesmo destaque colorido (`heroValorHTML`)
  dos outros ecrãs, em vez de um cartão com contorno.
- **Contraste (modo escuro):** os títulos dos ecrãs usam a cor clara do tema, com contraste de 9,7:1.
- **Proximidade:** o lucro do mês deixa de aparecer duas vezes em Estatísticas. Fica só no destaque do topo.

**Ainda por fazer (fora desta revisão):**
- Modo Dev é muito longo e tem muitas secções seguidas; pede uma reorganização em grupos.
- O botão "Escolher imagem" em Aparência tem pouco contraste no modo escuro.

## Versão 0.126 — Contraste dos botões da Aparência

- **Escolher imagem:** no modo escuro, o texto ficava na cor escura do tema, por causa da regra dos rótulos.
  Como este botão é uma etiqueta, agora mantém texto branco nos dois modos.
- **Remover:** no modo escuro usa o vermelho claro, para ter contraste sobre o fundo.
- O Modo Dev fica como está, porque é oculto.

## Versão 0.127 — Transição do conteúdo ao trocar Dia/Semana/Mês/Ano e de período

- Em Ganhos, Despesas, Lucro e Estatísticas, o conteúdo agora desliza como na troca de abas
  (`animarTroca`, com os mesmos keyframes `slideFromRight` e `slideFromLeft`).
- **Trocar de vista:** a direção segue a ordem Dia → Semana → Mês → Ano (para a frente, entra pela direita).
- **Navegar com as setas:** para a frente entra pela direita; para trás, pela esquerda.
- Aplica-se a Ganhos (cartões e lista), Despesas (destaque, fixas e variáveis), Lucro e Estatísticas.

## Versão 0.128 — Rebote nas trocas de aba e de seleção

- As trocas de ecrã (abas) e de conteúdo (Dia/Semana/Mês/Ano e setas) usam agora a mesma curva da gota,
  com uma pequena ultrapassagem que assenta (`cubic-bezier(.34,1.56,.64,1)`), em 0,42 s.

## Versão 0.129 — Quem nunca pagou fica só para leitura

- Antes, a conta sem assinatura ficava livre, e só expirar bloqueava. Agora segue a mesma regra das
  regras do Firestore: sem assinatura ativa, a conta fica em modo só leitura.
- Casos:
  - **A verificar** (ainda a ler a assinatura): não bloqueia.
  - **Sem assinatura** (nunca pagou): só leitura, com aviso "Sem assinatura ativa".
  - **Expirada:** só leitura, com aviso e data de expiração.
  - **Ativa:** acesso normal.
  - **Modo básico** (sem login): não se aplica; continua com os limites do modo básico.
- O cartão de assinatura deixou de dizer que "o acesso é livre".

## Versão 0.130 — Quem nunca pagou tem os direitos do modo básico

- Sem assinatura (nunca pagou), a conta passa a ter os mesmos direitos de quem não tem login:
  limite de 5 ganhos, 5 despesas variáveis e 5 despesas fixas, e as configurações bloqueadas
  (exceto IVA e comissão). Não envia dados para a nuvem.
- Quem tem a assinatura expirada continua só para leitura.
- Quem tem assinatura ativa tem acesso normal.
- O aviso de modo básico explica que é preciso ativar a assinatura para desbloquear tudo.
- Coerência com as regras do Firestore: sem `access/{uid}` as escritas são recusadas, e a app já não tenta escrever nesse caso.

## Versão 0.131 — Transparência 100% no Modo Dev

- Modo Dev → "Transparência 100% (efeito transparente)": quando ligado, cartões, barra e folhas ficam
  sem fundo próprio (opacidade 0), só com o contorno. Serve para testar o efeito.
- Só neste aparelho (`localSettings.glassFull`). Não altera o controlo de intensidade de Aparência.
- Testado: um cartão passa de 78% de opacidade para 0%.

## Versão 0.132 — Schema v1 como único formato de envio

- Todos os envios para a planilha (Apps Script) usam sempre o schema v1. Já não há interruptor
  "Gravar no formato novo" nem o botão "Converter e enviar agora" no Modo Dev.
- Nuvem (Firestore), cópia diária e exportação .json já usavam sempre o schema v1.
- **Ainda não feito:** o armazenamento local do aparelho e o modelo interno da app continuam no formato
  antigo (`bills`, `monthData`, `profile`). Eliminar a conversão exige reescrever o modelo interno.
  A leitura do formato antigo mantém-se para compatibilidade com dados já guardados no aparelho.

## Versão 0.133 — Remoção da transparência 100% e início da migração para v1

- Removido o botão "Transparência 100%" do Modo Dev (não era necessário).
- **Migração para schema v1 — fase 1:** `migrarLocalParaV1()` cria uma cópia v1 dos dados do aparelho em
  `stateV1` e guarda as chaves antigas em `backup_pre_v1_bills`, `backup_pre_v1_monthData` e
  `backup_pre_v1_profile`. Só corre uma vez. A app ainda lê o formato antigo nesta fase.
- Testado: totais de ganhos, contas e nome são iguais na cópia v1, e a migração não se repete.
- **Próximas fases:** ler sempre a cópia v1 nos cálculos e ecrãs, depois nas edições e na fusão com a nuvem,
  e só no fim remover a conversão.

## Versão 0.134 — Arranque sem Firebase

- Antes, a app só arrancava quando o Firebase carregava. Sem internet, ou com o Firebase bloqueado,
  a tela de carregamento ficava parada e a app não abria, mesmo com os dados guardados no aparelho.
- Agora, se o Firebase não estiver pronto em 3 segundos, a app arranca em modo local. Quando o Firebase
  chega (em aberturas seguintes), a sessão funciona normalmente.
- Testado sem rede: a app abre e mostra as abas, sem erros.

## Versão 0.135 — Schema v1 é a fonte de verdade (opção A)

- **Leitura:** na abertura, a app lê `stateV1` e monta a vista em memória (`bills`, `monthData`, `profile`)
  com `denormalizeFromSchemaV1`. O formato antigo só é lido se o v1 ainda não existir (fallback).
- **Escrita:** cada alteração grava o v1 (`salvarStateV1`). As chaves antigas (`bills`, `monthData`,
  `profile`) deixam de ser atualizadas e ficam só como cópia de segurança da migração.
- **Ordem no arranque:** migração primeiro, depois leitura.
- **Testado com dados completos:** contas (com dia/semana e override), ganhos com turno e km, despesas com
  litros, plataformas extra, metas e turnos. Os dados lidos são iguais aos originais, o lucro do mês
  bate com a conta feita à mão (253 €), uma edição fica gravada e persiste depois de reabrir.
- **Ainda por fazer:** a vista em memória continua no formato antigo. Só a fonte de verdade é v1.
  Se quiseres eliminar essa vista, é a opção B, que fica para depois.

## Versão 0.136 — Escolha clara entre dados da nuvem e do aparelho

- A folha "Dados diferentes" mostra dois cartões selecionáveis (nuvem e este aparelho), em vez de três
  botões com a mesma força visual.
- Ao tocar num cartão, ele fica destacado com um visto, e só então o botão de confirmar fica ativo,
  com a ação escrita no próprio botão. Antes de escolher, o botão fica cinzento.
- "Decidir depois" continua disponível e não altera nada.

## Versão 0.137 — Continuidade das barras do Safari com o efeito transparente

- No Safari (sem estar instalada no ecrã inicial), a barra de cima e a de baixo são pintadas com uma
  cor sólida, definida por `theme-color`. A app não consegue deixar o gradiente de fundo subir até essas barras.
- Com o efeito transparente, a cor dessas barras passa a ser o topo do gradiente (creme no claro, escuro no
  escuro), para ficar o mais próximo possível.
- **Limite:** no Safari a continuidade total não é possível. Com a app instalada no ecrã inicial, o conteúdo
  pode passar por baixo da barra de estado (`black-translucent`), e aí a continuidade fica completa.

## Versão 0.138 — Fundo do efeito transparente sob a zona inferior (iOS instalado)

- No iPhone com a app instalada no ecrã inicial, havia uma faixa vazia na parte de baixo, sob o
  indicador do ecrã. A camada de gradiente de fundo ficava limitada à janela visível.
- Agora essa camada é estendida 160 px para além da borda inferior, por isso cobre também essa zona.
- Verificado no navegador que a camada passa a ultrapassar a janela. Não testado no iPhone real.

## Versão 0.139 — As duas opções da folha de conflito têm o mesmo peso

- O botão de confirmar da nuvem usava o estilo de texto vermelho (de perigo), e parecia um erro. Agora
  as duas opções usam o mesmo botão sólido, com texto neutro: "Usar os dados da nuvem" e "Usar os dados
  deste aparelho".
- O texto da folha indica que o outro conjunto é substituído, sem marcar nenhuma opção como errada.

## Versão 0.140 — Ajustes para iOS (ícone, pop-up e barra de abas)

- **Ícone do ecrã inicial:** o ícone do iPhone tinha transparência e aparecia com uma faixa preta. Foi gerado
  um novo ícone opaco (carro dourado sobre fundo verde), sem zonas transparentes.
- **Pop-up:** passa a ficar abaixo da barra de estado (`env(safe-area-inset-top)`), e deixa de ficar escondido e
  desfocado no topo.
- **Barra de abas:** estendida 240 px para baixo com a cor de fundo, para chegar ao fim do ecrã, incluindo a
  zona do indicador. No efeito transparente, a extensão fica transparente, e a camada de fundo cobre o resto.
- **Fundo:** o `html` passa a ter a cor de fundo da app, para não aparecer uma faixa bege.
- Testado no navegador; não testado num iPhone real.

## Versão 0.141 — Ícones: iOS sem cantos próprios, Android com margem

- **iOS:** o ícone do ecrã inicial passa a ser um quadrado cheio, sem cantos arredondados. O iOS aplica a
  sua própria máscara; com cantos próprios, apareciam cantos estranhos. O carro fica centrado, com margem.
- **Android:** o manifest passa a ter dois ícones. O normal (arredondado) e um "maskable" com o carro reduzido
  para cerca de 60%, dentro da margem de segurança. Assim o carro deixa de parecer muito grande.
- Ícones gerados e verificados visualmente. Não testados no iPhone nem em Android reais.
- Ao contrário das alterações na app, o ícone só muda depois de reinstalar a app no ecrã inicial.

## Versão 0.142 — Ícone Android arredondado, com o carro reduzido

- O ícone "maskable" (quadrado, sem cantos) fazia a tela de arranque e o multitarefas do Android aparecerem quadrados.
- O manifest volta a ter um único ícone arredondado, com o carro reduzido dentro do próprio bloco
  (cerca de 60%). Mantém os cantos arredondados e o tamanho certo.
- O ícone do iOS continua quadrado cheio, porque o iOS aplica o seu próprio arredondamento.

## Versão 0.143 — Carro maior no ícone do iOS

- Na tela "Adicionar à Tela de Início" do iPhone, o carro aparecia pequeno e com muita margem.
  O ícone do iOS passa a ter o carro maior (cerca de 90% da escala anterior de desenho), mantendo o quadrado cheio.
- O ícone do Android não foi alterado.

## Versão 0.144 — Ícone do iOS com o mesmo desenho da app

- O carro do ícone do iOS tinha sido redesenhado à mão, e as rodas e os retrovisores ficaram com
  proporções diferentes, o que dava ar de deformação.
- O ícone do iOS passa a ser gerado com as mesmas coordenadas do ícone SVG da app (teto, corpo,
  retrovisores e rodas), num quadrado cheio, para ficar igual ao que já está na tela de carregamento.

## Versão 0.145 — Ícone do iOS segue a cor escolhida, antes de instalar

- O preview de "Adicionar à Tela de Início" no iPhone lê o ícone do HTML (apple-touch-icon), que era fixo
  (verde por omissão).
- Agora `atualizarIconeIOS()` gera esse ícone num canvas, com a cor escolhida em "Cor do ícone" e o contorno
  do efeito transparente, e atualiza o link sempre que essas opções mudam.
- Testado no navegador: roxo e azul com contorno aparecem corretamente.
- **Limites:** o iOS lê o ícone no momento de adicionar. O Android continua a usar o manifest, que é fixo
  na instalação, por isso o ícone instalado no Android só muda ao reinstalar.

## Versão 0.146 — Ícone do Android (manifest) segue a cor escolhida

- O ícone do manifest passa a ser atualizado com a cor escolhida em "Cor do ícone" e com o efeito transparente,
  como já acontece com o ícone do iOS.
- O Chrome lê o manifest ao oferecer a instalação, por isso o ícone que aparece na instalação deve seguir a escolha.
- **Limite:** o ícone já instalado no ecrã inicial, e o do multitarefas, só mudam ao reinstalar a app.
- Testado no navegador: mudar a cor altera o ícone do manifest. Não testado no Android real.

## Versão 0.147 — Uma só linha antes do lucro líquido (Lucro)

- No detalhamento do Lucro, havia duas linhas seguidas antes do total: a linha inferior de "Faturamento bruto"
  e a linha superior mais grossa do total. Agora fica só uma linha separadora.

## Versão 0.148 — Modo Dev organizado por assunto

- **Sincronização:** Sincronização (URL), Migrar planilha, Encriptação e Log de sincronização.
- **Dados:** Dados ocultos e limpeza, Importar dados.
- **Aplicação:** Atualização.
- **Extras:** Mensagem de aniversário, Brincadeira secreta e Resumo de ontem.
- Só a ordem mudou; nenhuma opção foi alterada. Estrutura de `<div>` conferida.

## Versão 0.149 — Atualização verificada antes do conteúdo

- **Antes:** a app mostrava o conteúdo e só depois verificava a versão. Quando havia uma versão nova, voltava
  à tela de carregamento para atualizar, e carregava duas vezes.
- **Agora:** a verificação acontece no início do arranque, antes de mostrar o conteúdo. Enquanto verifica, a tela
  de carregamento mostra "A verificar atualização…" com uma barra de progresso. Se houver versão nova, a app
  recarrega uma única vez e abre já atualizada.
- Tempo limite de 2,5 s: sem rede, a app abre logo, sem esperar.
- **Sem sessão (modo básico):** continua a não recarregar sozinha; mostra o alerta depois de abrir, com a opção de entrar.
- Testado com um servidor local e uma versão simulada: um único recarregamento, sem mostrar o conteúdo antigo.

## Versão 0.150 — Resumo de ontem ativo por omissão e texto da tela de carregamento suave

- **Resumo de ontem:** passa a vir ligado por omissão (`dailySummaryEnabled` → `true`). Quem já tinha escolhido
  explicitamente continua com a sua escolha, guardada no perfil.
- **Tela de carregamento:** as mensagens (ex.: "A carregar…", "A verificar atualização…") deixam de piscar. O texto
  sai com desvanecimento, muda e volta a entrar (`trocarTextoSplash`, 0,25 s por fase). Pedidos seguidos não se atropelam.
- Testado: resumo ativo por omissão; a meio da troca a opacidade fica intermédia, e no fim o texto novo aparece.

## Versão 0.151 — Criptografia de dados, fase 1 (núcleo, sem ligação à sincronização)

Modelo: uma chave de dados (DEK) de 256 bits, aleatória, cifra os campos. A DEK fica guardada três vezes,
cada uma embrulhada com AES-GCM, com uma chave derivada de:
1. a senha (mín. 8 caracteres, com maiúscula e caractere especial);
2. a chave de recuperação de 24 caracteres (gerada pela app, sem caracteres ambíguos);
3. as respostas às 4 perguntas de recuperação (normalizadas: sem maiúsculas nem acentos).

Trocar a senha não obriga a recifrar os dados. Quem tiver a chave de recuperação ou as respostas recupera o acesso.

Funções: `criarCofreCripto`, `abrirCofreCripto`, `cifrarCampo`, `decifrarCampo`, `gerarChaveRecuperacao`.
PBKDF2-SHA256 com 600 000 iterações.

**Riscos:** as respostas funcionam como segunda chave. Respostas comuns podem ser testadas offline se a base de
dados vazar. A app deve exigir respostas difíceis de adivinhar (mín. 12 caracteres no total) e avisar isto.

Testado no navegador: formato da chave, recuperação pela senha, pela chave e pelas respostas, rejeição de senha
e respostas erradas, senha fraca recusada, e cifra/decifra de campos.
Ainda não ligado à interface nem à sincronização.

## Versão 0.152 — Criptografia dos dados, fase 2 (ecrã de configuração)

- **Entrada:** Você → "🔐 Criptografia dos dados", abaixo do cartão de conta.
- **Passos:** aviso de riscos e responsabilidade (botão só ativa após marcar a caixa) → senha (confirmada duas vezes,
  mín. 8 caracteres, com maiúscula e caractere especial) → 4 perguntas escritas pelo utilizador, com resposta livre
  (texto ou data) → chave de recuperação com QR code, guardar imagem ou partilhar.
- **Respostas livres:** sem tamanho mínimo; só não podem ficar vazias. O aviso sobre respostas fáceis de adivinhar
  continua no ecrã.
- **Cofre:** guardado em `cofreCripto` neste aparelho. Não contém chaves em claro.
- **Apagar dados ou tudo** remove também o cofre, porque deixa de haver dados para proteger.
- **Imagem:** contém o QR, a chave e o aviso de responsabilidade. Pode ser guardada ou partilhada.
- **QR code:** carregado de cdnjs (qrcodejs). Sem ligação, a chave aparece escrita, com a mensagem de fallback.
- **Ainda não feito:** leitura do QR com a câmara ou a galeria (fase 3), lembrar a chave no aparelho (fase 3),
  remover a criptografia (fase 4) e ligação à sincronização (fase 4).
- Testado no navegador: aviso, senha, perguntas com texto e data, criação do cofre, imagem gerada e remoção do cofre
  ao apagar dados. A criação demora alguns segundos, porque a derivação usa 600 000 iterações.

## Versão 0.153 — Criptografia dos dados, fase 3 (aparelhos, QR e recuperação)

- **Lembrar neste aparelho:** ao criar ou desbloquear, a chave de dados (DEK) fica guardada em `dekAparelho`.
  Por omissão, lembra. Desmarcando "Lembrar neste aparelho", a chave só vale nessa sessão.
- **Aparelho novo:** ao abrir Criptografia dos dados sem chave neste aparelho, pede uma de quatro formas:
  senha; chave de recuperação escrita; QR pela câmara (`cripLerQrCamera`); QR escolhido na galeria (`cripLerFicheiro`).
- **Esqueci a senha:** responde às 4 perguntas (respostas de data aparecem como campo de data). Se conferirem,
  define uma nova senha. A chave de dados não muda, só volta a ser embrulhada com a nova senha (`trocarSenhaCofre`).
- **Esquecer a chave neste aparelho:** no ecrã de criptografia ativa, remove `dekAparelho`.
- **Apagar dados ou tudo:** remove também `dekAparelho`.
- **Cofre:** passa a guardar o tipo de cada pergunta (texto ou data), para a recuperação mostrar o campo certo.
- **Leitor de QR:** jsQR, do cdnjs, carregado só quando é preciso.
- Testado no navegador: senha certa e errada, chave certa e errada, lembrar e não lembrar, QR lido a partir de imagem,
  recuperação pelas perguntas (com data) e nova senha, e esquecer a chave.
- **Não testado automaticamente:** a câmara. Precisa de um telemóvel real: abre Criptografia, Desbloquear,
  "Ler QR com a câmara" e confirma que a câmara abre e lê a imagem guardada.

## Versão 0.154 — Criptografia dos dados, fase 4 (ligação à sincronização)

- **Campos cifrados na nuvem:** valores (ganhos, despesas, despesas fixas e pagamentos), notas, litros, nomes das
  despesas fixas e nomes das plataformas extra. Ficam dentro de `enc` em cada registo, cifrados com a chave de dados.
- **Em claro:** datas, identificadores, categorias, tipos, plataformas Uber/Bolt, chaves das plataformas extra
  (aleatórias, sem o nome), e o bloco público do cofre (`crypto`, com os sais e as chaves embrulhadas).
- **Plataformas extra:** ao criar o cofre, as chaves passam a ser aleatórias (`custom_` + id), para o nome não aparecer
  na chave. Os registos antigos são atualizados localmente. Ao criar uma plataforma, dedupe pelo nome.
- **Escrita:** `prepararParaNuvem` cifra o documento antes de gravar. Sem chave neste aparelho, não grava nada.
  Nunca se grava em claro com criptografia ativa.
- **Leitura:** `prepararNuvem` decifra. Se o aparelho não tiver a chave, pede o desbloqueio e a sincronização para.
  Um aparelho novo guarda o bloco `crypto` da nuvem como cofre local, e depois pede a senha, a chave ou o QR.
- **Remover criptografia:** pede a senha, grava os dados em claro (com `pushToFirestore(true)`) e apaga o cofre local.
  Os outros aparelhos vêem a nuvem sem `crypto` e apagam o seu cofre (marca `naNuvem`).
- **Cópia diária na planilha:** não corre com criptografia ativa, para não enviar dados em claro.
- **Criar cofre:** se a nuvem já tiver criptografia, não cria outra: abre o desbloqueio (`window.__fbCloudCrypto`).
- **Regras do Firestore:** `crypto` foi acrescentado à lista de chaves permitidas em `hasValidShape()`.
  As regras ainda não estão publicadas; devem ser publicadas com este ficheiro.
- Testado com uma nuvem simulada: nada sensível em claro, aparelho novo pede desbloqueio e recupera os dados, remoção
  devolve os dados em claro e apaga o cofre local.

## Versão 0.155 — A foto de perfil fica só neste aparelho

- A foto de perfil deixa de ir para a nuvem (e para a cópia de segurança). Antes seguia como texto dentro do documento,
  o que pesava no limite de tamanho do Firestore.
- O nome continua a sincronizar normalmente.
- `applyState` mantém a foto local quando chegam dados da nuvem sem foto.
- Os documentos antigos que ainda tinham a foto deixam de a ter na próxima gravação.
- Testado: a foto não entra no documento da nuvem, o nome entra, e aplicar dados da nuvem não apaga a foto local.

## Versão 0.156 — Arquivo compactado na nuvem e aviso de limite

**Arquivo:** os meses com mais de 12 meses saem do documento principal e ficam num bloco único, comprimido com gzip
(`CompressionStream`). Se houver criptografia, o bloco também é cifrado (AES-GCM com a chave de dados).
- Escrita: `arquivarAntigos` separa os registos; `empacotarArquivo` comprime (e cifra, se aplicável).
- Leitura: `prepararNuvem` descompacta o bloco e junta-o aos dados recentes, antes de converter. A fusão entre
  aparelhos não muda, porque o aparelho continua com o histórico completo.
- Sem `CompressionStream` (navegadores antigos), nada é arquivado e o documento segue completo.
- **Limitação:** a descompactação acontece ao sincronizar, e não só quando o utilizador abre um mês antigo. Assim, a
  fusão não precisa de lógica especial. Um arquivo consultado sob demanda exigiria mudar a forma como a fusão
  trata meses ausentes.

**Aviso de limite:** antes de gravar, o documento é medido. Acima de 1 000 000 bytes, a gravação não é feita, aparece
uma folha a explicar o que aconteceu (os dados ficam neste aparelho) e fica registo no log de sincronização.
Acima de 80% do limite, aparece um aviso de espaço quase cheio, uma vez por sessão. Falhas de gravação passam a
mostrar uma mensagem, em vez de apenas "offline / sem link".

**Regras:** `arquivo` foi acrescentado à lista de chaves permitidas em `hasValidShape()`, junto de `crypto`.

Testado com uma nuvem simulada (30 meses, 1 200 ganhos):
- O documento principal fica com 12 meses; o documento total fica com cerca de 65 KB.
- Ler de volta devolve os mesmos registos, com e sem criptografia.
- Com dados acima do limite, a gravação é recusada e o aviso aparece.

## Versão 0.157 — Folhas: fundo mais opaco, fecho animado, alça arrastável; resumo com o último dia

- **Efeito transparente:** o fundo das folhas passa de 82% para 96% de opacidade (claro e escuro), para o texto por
  trás não atrapalhar a leitura.
- **Fecho animado:** a folha desce e desaparece (260 ms), e o fundo esmaece. Vale para todas as folhas. Abrir outra
  antes do fim do fecho cancela o fecho pendente (`closeTimer`).
- **Alça arrastável:** arrastar a alça para baixo acompanha o dedo. Passados 90 px, ou com um gesto rápido, a folha
  fecha. Se o arrasto for curto, volta ao lugar. A área de toque da alça é maior.
- **Resumo de ontem sem registos:** mostra "Não houve registos ontem." e, abaixo, o último dia com registos
  (`ultimoDiaComRegisto`), com os mesmos cartões. Continua a aparecer só uma vez por dia.
- Testado: fecho animado, arrastar para fechar, arrasto curto que volta ao lugar, resumo sem registos com o último
  dia, e opacidade de 96% nos dois modos.

## Versão 0.158 — Idioma: português e inglês (primeira parte)

- **Escolha:** em Você → "Idioma" (PT / EN), e na tela de boas-vindas, antes do login, que aparece na primeira
  abertura. A escolha fica em `localSettings.lang` (neste aparelho, não sincroniza).
- **Dicionário:** `I18N` (pt/en), com `tr('chave')`. Textos fixos do HTML têm `data-i18n`; `aplicarIdioma()` atualiza-os.
- **Datas e números:** `LOCALE()` devolve `pt-BR` ou `en-GB`, em todos os formatos de data.
- **Já traduzido:** separadores, títulos principais, Dia/Semana/Mês/Ano, estado da sincronização, texto do login,
  boas-vindas, Idioma, Conta, Preferências, Configurações e Aparência.
- **Ainda em português (segunda parte):** folhas e mensagens (toasts), estatísticas, ecrãs de detalhe, Aparência,
  e a maioria dos textos de Configurações.
- Testado: separadores e títulos em inglês, datas em inglês, boas-vindas trocam de idioma, idioma guardado.
  Corrigida uma recursão no `LOCALE()` causada pela substituição de `'pt-BR'`.

## Versão 0.159 — Idiomas em ficheiros separados

- O português fica em `corridaplus.html` (`I18N.pt`). Cada outro idioma fica num ficheiro próprio, carregado só quando é
  escolhido: inglês em `idioma-en.js`.
- `IDIOMA_FICHEIROS` diz qual ficheiro corresponde a cada código. `carregarIdioma(lang)` injeta o script, com `?v=<versão>`
  para o navegador não usar uma tradução antiga.
- **Adicionar um idioma:** copiar `idioma-en.js`, mudar o código e traduzir os valores; depois acrescentar o código em
  `IDIOMA_FICHEIROS` e uma opção no seletor de Você e nas boas-vindas.
- **Ficheiro em falta:** a app continua em português e mostra "Não foi possível carregar o idioma".
- **Publicação:** `idioma-en.js` tem de ficar na mesma pasta que `corridaplus.html` (GitHub Pages).
- Testado: trocar para inglês carrega o ficheiro; ao recarregar com inglês guardado, o ficheiro volta a carregar; com
  o ficheiro em falta, fica em português.

## Versão 0.160 — Francês

- Novo ficheiro `idioma-fr.js`, com os mesmos textos já traduzidos para inglês. Datas em `fr-FR` (`LOCALES`).
- Acrescentado às escolhas de Você e das boas-vindas, e a `IDIOMA_FICHEIROS`.
- Testado: separadores, datas e seletor em francês; o francês continua depois de recarregar.

## Versão 0.161 — Tradução completa, passo 1: Ganhos e Despesas

- Textos de Ganhos e Despesas (títulos, botões, folhas, avisos e mensagens) passam a usar chaves `tr('...')`.
- O português fica em `I18N.pt` no ficheiro principal. Inglês e francês ficam em `idioma-en.js` e `idioma-fr.js`.
- Os cabeçalhos das colunas da exportação para Excel não foram alterados (são nomes de colunas do ficheiro).
- Ainda por traduzir neste passo: "💰 Lucro líquido" e "⚠️ Faltou" (com emoji, dentro de modelos), que ficam para o passo 2.
- Testado: Ganhos e Despesas em inglês, francês e português; mensagens em inglês.

## Versão 0.162 — Tradução completa, passo 2: Lucro e Estatísticas

- Títulos, cartões, gráficos, legendas, metas, análise do mês e do ano, melhores dias, ganhos por turno e
  detalhamento do lucro passam a usar chaves `tr('...')`.
- Frases com valores (ex.: "Faltam X para a meta", "Total no fim do mês: X") montadas com `tr()` à volta do valor.
- Os textos com emoji (ex.: "🏆 Melhor dia", "📊 Resumo do mês") mantêm o emoji e trocam só o texto.
- Ficheiros de idioma atualizados: `idioma-en.js` e `idioma-fr.js` (171 chaves cada).
- Testado: Lucro e Estatísticas em português, inglês e francês, com verificação de que cada idioma mostra as suas palavras.
- Ainda por traduzir: os dias da semana abreviados (Seg, Ter, …), que estão numa constante carregada antes do idioma;
  e os textos de Configurações, Aparência, folhas de criptografia e mensagens de sincronização (passos 3 e 4).

## Versão 0.163 — Tradução completa, passo 3 (parte a): Você e Aparência

- Títulos, rótulos, botões e mensagens curtas de Você e Aparência usam chaves `tr('...')` (46 chaves).
- Cabeçalhos com maiúsculas (ex.: "IMPOSTOS") são estilo visual: o texto guardado continua em maiúsculas/minúsculas normais.
- Corrigida a tradução de "Impostos" em francês ("Impôts").
- Testado: Você em português, inglês e francês.
- **Ainda por traduzir (parte b):** as descrições longas de Configurações (ex.: "Liga só o que usas…", "Desconta o IVA…"),
  "Repor turnos padrão" e outros textos de turnos, as folhas de criptografia, de exportação e de apagar dados,
  a lista de sincronização e os textos da Aparência com descrições longas.

## Versão 0.164 — Tradução completa, passo 3 (parte b): descrições e apagar dados

- Descrições longas de Configurações e Aparência, a secção de modo básico, os textos de apagar dados e os textos de turnos
  passam a usar chaves `tr('...')` (28 chaves novas).
- Correção: o texto "Modo básico." dentro de uma string com aspas simples passou a concatenação (`+ tr(...) +`), para não
  quebrar o script.
- Dicionários: 245 chaves em cada idioma.
- Testado: a descrição de Configurações nos três idiomas.
- **Ainda por traduzir (passo 4):** cerca de 220 textos de mensagens, folhas de criptografia (passos, avisos, imagem da chave),
  exportação, sincronização e avisos de atualização. A lista está em `/tmp/cands.json` na sessão de trabalho.

## Versão 0.165 — Tradução completa, passo 4: mensagens, folhas e avisos

- Mensagens (toasts), folhas de criptografia, exportação, sincronização, atualização, limites da nuvem e botões
  passam a usar chaves `tr('m_...')` (188 chaves novas, com inglês e francês).
- Textos com aspas simples que contêm uma chave passam a concatenação (`' + tr('...') + '`), para não quebrar o script.
- O dicionário `I18N` passa para o início do script. Com isso, o carregamento não depende da ordem das declarações.
- Correção: o texto "Entrar com Google" também estava dentro do próprio dicionário e foi trocado por `tr()`; foi reposto.
- Verificação: todas as 426 chaves usadas no código existem nos três dicionários.
- Testado: ecrãs principais (Ganhos, Despesas, Lucro, Estatísticas, Você) e folhas de criptografia, reset e exportação
  nos três idiomas, sem erros.
- **Ficam por traduzir:** os textos com valores variáveis que ainda estão montados à mão (ex.: "Desfoque com granulação:"
  com o valor depois), e os dias da semana abreviados (Seg, Ter…).

## Versão 0.166 — Tradução: restos (dias da semana e textos com valores)

- Dias da semana abreviados lidos no momento de uso (`weekdaysShort()`), em `wd_seg` … `wd_dom`:
  Seg/Ter/Qua/Qui/Sex/Sáb/Dom, Mon…Sun, Lun…Dim.
- Textos com dois-pontos e valor montados à mão (fundo da imagem: véu e desfoque; erro de atualização;
  versão no email de suporte) passam a usar `tr()`.
- Verificação: todas as chaves usadas no código existem nos três dicionários.
- Testado: dias da semana nos três idiomas; ajuste do véu da imagem sem erros.

## Versão 0.167 — Barra de progresso fica na tela de carregamento até ao fim

- **Antes:** a barra aparecia só durante a verificação de atualização, e era escondida logo a seguir, mesmo que a
  sincronização ainda estivesse a decorrer.
- **Agora:** a barra começa visível com a tela de carregamento e só desaparece quando a tela sai (no `hideSplash`).
  Isto cobre a verificação, a sincronização e a mensagem final, sem piscar.
- Testado: a barra fica visível em todas as amostras do carregamento.

## Versão 0.168 — Tradução: assinatura, conta, totais e metas

- Aviso de assinatura expirada (modo só leitura), cartão da assinatura, mensagens de período de teste, cancelamento e
  dias restantes, passam a usar chaves `tr('m_...')`.
- Cartão da conta (conectado com Google, "Sair", mostrar/ocultar, "trocar", "Assinatura"), totais do Ganhos ("Total bruto",
  "TVDE") e o cabeçalho do Lucro ("Lucro líquido") passam a traduzir.
- Frase do mês ("recebi em … para usar em …") traduzida.
- Descrição longa das metas de lucro passa a `data-i18n` (estava no HTML estático).
- Verificação: 455 chaves usadas, todas presentes nos três dicionários.
- Testado com assinatura expirada em inglês: aviso, Lucro, Ganhos e Configurações sem palavras em português.
- Termo mantido: "TVDE" (sigla do regime), com "VTC" em francês.

## Versão 0.169 — Aviso de assinatura no computador

- No ecrã largo, o aviso de assinatura (modo só leitura) era inserido na primeira célula da grelha, ao lado da barra
  lateral, e ficava por cima do menu. Agora tem `grid-column:2`, e alinha com o conteúdo.
- No telemóvel, o comportamento não muda (o `grid-column` é ignorado em layout de coluna única).
- Testado: 1440 px e 390 px, o aviso começa à mesma posição que o conteúdo.

## Versão 0.170 — Aviso de assinatura com fundo sólido

- O aviso de modo só leitura tinha fundo a 14% de opacidade, e o conteúdo por trás aparecia por cima do texto.
- Agora tem fundo sólido: claro (#F7E6DF) e escuro (#3A2520), com ou sem o efeito transparente.
- Testado nos quatro modos (normal/transparente × claro/escuro): fundo totalmente opaco.

## Versão 0.171 — Aviso de assinatura atualiza ao mudar de idioma

- O aviso de modo só leitura e o cartão da assinatura eram desenhados uma vez e não eram atualizados ao mudar de idioma.
  `setIdioma` passa a chamar `renderReadOnlyBanner()` e `renderSubscriptionCard()`.
- Testado: em inglês o aviso fica "Subscription expired on 13 October 2026…"; ao voltar ao português, volta ao texto anterior.

## Versão 0.172 — Gravação na nuvem sem campos undefined

- **Erro:** `invalid-argument` / "Unsupported field value: undefined". O Firestore recusa qualquer campo com valor
  `undefined`, e a gravação falhava por inteiro. Sem gravação, as alterações ficavam só no aparelho.
- **Correção:** antes de gravar, o documento passa por `JSON.parse(JSON.stringify(...))`, que remove os campos `undefined`.
  Essa versão limpa é a que é medida e enviada.
- **Diagnóstico:** `caminhosUndefined` lista onde existia o campo, e o registo de sincronização mostra
  "Campos vazios removidos antes de gravar" com até 5 caminhos. Assim é possível ver a origem.
- Testado: função encontra os caminhos em amostra com `undefined`; versão limpa fica sem nenhum.

## Versão 0.173 — Folhas mais opacas e mais traduções

- As folhas ficavam transparentes porque a opacidade acompanhava a intensidade do efeito transparente
  (com "Mais transparente" caía para cerca de 38%). Agora é fixa em 96%, claro e escuro.
- Traduzidas as telas visíveis nas capturas: nova receita (período e IVA), nova despesa fixa (nome, frequência e texto),
  turnos (Emoji e Nome), Aparência (intensidade, transparente/padrão/opaco e imagem de fundo) e a folha de criptografia
  (os sete parágrafos).
- Testado em inglês: receita, despesa fixa, Aparência e criptografia sem palavras em português.
- Ainda por fazer: varredura completa dos textos restantes do código.

## Versão 0.174 — Varredura de traduções

- Textos restantes do script passam a usar chaves: avisos de assinatura e de limite, conflito de dados, QR e chave
  de recuperação, exportação em JSON, reposição (dados, configurações, apagar tudo), migração, falhas de nuvem e de
  biblioteca de exportação, e o aviso de espaço na nuvem.
- Corrigida uma chave dentro de aspas simples (`' + tr(...) + '`) que partia o script.
- Dicionários: 506 chaves, todas presentes em pt, en e fr.
- Testado em inglês e francês: cripto, exportação, limite e dados sem palavras em português.
- **Login Google:** a chave de API tem de aceitar pedidos do domínio de autenticação do Firebase
  (`tvdecorridas.firebaseapp.com`) e a app tem de estar em "Domínios autorizados" no Firebase Authentication.

## Versão 0.175 — Botão "Renovar assinatura" acessível

- **Causa:** com a assinatura expirada, a folha do resumo de ontem abria por cima do aviso e cobria o botão. O primeiro
  toque caía na folha (que a fechava), e o botão parecia não fazer nada.
- **Correção:** a folha do resumo é marcada (`window._sheetKind`). Quando o aviso de expiração é desenhado, essa folha
  fecha-se sozinha. O resumo de ontem já não abre por si quando a assinatura está expirada.
- Testado: resumo aberto, assinatura expira, resumo fecha, e o botão leva às Configurações.
- **Ainda não faz pagamento:** o botão só leva às Configurações. O checkout do Stripe depende do plano Blaze (ver roteiro).

## Versão 0.176 — Renovação P2P (provisória, até ao plano Blaze)

- O botão "Renovar assinatura" (aviso de modo só leitura) abre a folha de planos: 1 mês 10 €, 3 meses 25 €,
  6 meses 50 € (`PLANOS_RENOVACAO`).
- Ao escolher um plano, aparecem os dados de pagamento (`PAGAMENTO_P2P`: MB WAY, IBAN e titular) e um botão que
  envia o comprovativo por email, com plano, valor, email e UID já preenchidos.
- **Dados de pagamento por preencher** em `PAGAMENTO_P2P` antes de publicar.
- **Ativação manual:** a app só desbloqueia quando existe `customers/{uid}/subscriptions/{id}` com `status`
  `active` (ou `trialing`) e `current_period_end` no futuro. Até ao Blaze, essa escrita é feita à mão na consola do Firebase.
- Testado: três planos, valor de 3 meses (25 €), dados de pagamento e email com UID e valor.

## Versão 0.177 — Gráficos de pizza

- Nova função `pieSVG(fatias)`: fatias em SVG, com legenda que mostra percentagem e valor. Cores em `PALETA_PIZZA`.
- Estatísticas do mês: "Despesas por categoria em pizza" (logo abaixo das barras) e "Distribuição do lucro por semana"
  (só semanas com lucro positivo).
- Estatísticas do ano: "Despesas por categoria em pizza".
- Títulos novos em pt, en e fr (`lucro_distrib`, `despesas_distrib`).
- Testado em inglês: as fatias das despesas aparecem e os títulos estão traduzidos.

## Versão 0.178 — Meta mínima e meta alvo

- Nas Configurações, cada período (mensal e semanal) tem agora duas metas: **mínima** (`monthlyMin`, `weeklyMin`)
  e **alvo** (`monthlyGoal`, `weeklyGoal`, a meta de lucro que já existia).
- Sem meta mínima definida, o mínimo continua a ser as despesas fixas do período.
- Sem meta alvo, o alvo passa a ser o mínimo.
- A barra de progresso mostra um traço no ponto do mínimo. A cor muda: vermelho abaixo do mínimo, dourado entre o
  mínimo e o alvo, verde ao atingir o alvo. O texto diz quanto falta para o próximo objetivo.
- Traduzido em pt, en e fr.
- Testado: lucro de 40 € com mínimo de 50 € e alvo de 200 € mostra "abaixo do mínimo · faltam 10 € para o mínimo".

## Versão 0.179 — Aviso de assinatura na área segura do iPhone

- No iPhone, o aviso de modo só leitura ficava colado ao topo (`top: 0`), por baixo da barra de estado (hora, bateria).
  O conteúdo que passava por trás aparecia desfocado por cima do aviso.
- Agora o aviso fica em `top: env(safe-area-inset-top)`, logo abaixo da barra de estado.
- Uma faixa fixa (`body::after`) com a cor do fundo cobre a área da barra de estado, para o conteúdo não aparecer por trás.
- Não testado num iPhone real: no teste de computador a área segura é zero.

## Versão 0.180 — Cores das categorias nos gráficos

- As despesas por categoria tinham a mesma cor fixa (vermelho), por isso a pizza ficava toda da mesma cor.
- Agora cada categoria tem a sua cor, fixa pela ordem em `EXPENSE_CATEGORIES` (`corCategoria`), nas barras e na pizza.
- Testado: três categorias com três cores diferentes na pizza.

## Versão 0.181 — Espaço acima do aviso de assinatura

- O aviso de modo só leitura encostava ao topo do ecrã, logo abaixo da barra de estado do iPhone.
- Agora tem 14 px de margem acima, somados à área segura (`env(safe-area-inset-top)`), também no `top` quando o ecrã rola.
- Testado: 14 px de folga acima do aviso no teste de computador. No iPhone, a folga é a área segura mais 14 px.

## Versão 0.182 — Banner para instalar a app (modo web)

- Quando a app está no navegador (não instalada), aparece um banner no topo com "Instalar a app".
- **Android:** o botão "Instalar" chama o pedido de instalação do navegador (`beforeinstallprompt`).
- **iPhone:** não existe esse pedido. O botão "Como instalar" abre um passo a passo: Partilhar → Adicionar ao ecrã
  principal → Adicionar. O mesmo guia tem a versão de Android (menu ⋮ → Instalar app).
- Os ícones dos passos são desenhos simples em SVG, não capturas de ecrã reais.
- O banner pode ser dispensado: não volta durante 14 dias (`instalarDispensadoEm`).
- Traduzido em pt, en e fr.
- Testado: banner na versão web, três passos com ícones, botão "Instalar" com o evento do Android, dispensa do banner.

## Versão 0.183 — Remove a faixa de cor no topo (iPhone)

- A faixa `body::after` (versão 0.179), com cor sólida, tapava a área da barra de estado. No iPhone aparecia como uma
  faixa de cor diferente do fundo do ecrã, sobretudo com o efeito transparente.
- Removida. O fundo segue a tela até ao topo, como no resto do ecrã. O conteúdo que passa por baixo da barra de estado
  continua a ser o comportamento normal do iOS.
- Testado: a faixa deixou de existir no CSS.

## Versão 0.184 — Tutorial de instalação legível no modo escuro

- Os ícones dos passos usavam a cor escura do tema (`--vault`) sobre um fundo escuro, por isso ficavam invisíveis.
  Agora usam `--ink`, que é clara no modo escuro e escura no claro.
- Testado: no modo escuro, o ícone fica claro sobre o fundo escuro.

## Versão 0.185 — Barra de estado igual à tela de carregamento

- No iPhone, a barra de estado toma a cor da tag `theme-color`. No arranque, essa tag ainda tinha a cor de tema por
  omissão, por isso aparecia uma faixa de outra cor no topo, que só mudava depois de a app aplicar o tema.
- Agora a tag `theme-color` (claro e escuro) recebe a mesma cor da tela de carregamento logo ao carregar a página.
- A cor é calculada pelo navegador (o fundo pode ser uma expressão `color-mix`) e convertida para hexadecimal.
- Testado: no modo escuro, `theme-color` fica `#121b19`, igual ao fundo da tela de carregamento.

## Versão 0.186 — Renovação só com o plano de 1 mês

- `PLANOS_RENOVACAO` tem agora só o plano de 1 mês (10 €). A folha de renovação mostra logo os dados de pagamento,
  sem lista de planos.
- O email de destino (`EMAIL_COMPROVATIVO`) aparece escrito no ecrã, e o botão abre o email com o assunto, o plano,
  o valor, o email e o UID já preenchidos.
- Testado: sem botões de plano, 10 € visível, email de destino visível e o link `mailto:` com o UID e o valor.

## Versão 0.187 — Aviso de renovação antecipada (5 dias)

- Com a assinatura ativa e 5 dias ou menos até ao fim, aparece um aviso no topo: "Faltam N dias da assinatura" com o
  botão "Renovar assinatura", que abre a folha de renovação de 1 mês (10 €).
- `renderRenovarAviso()` é chamada por `renderReadOnlyBanner()`, que já é atualizada quando a assinatura muda e a cada minuto.
- Com 6 dias ou mais, ou depois de expirar, o aviso não aparece (depois de expirar, vale o aviso de modo só leitura).
- Testado: 3 e 5 dias mostram o aviso; 6 dias e expirada não; o botão abre a renovação.
- Nota: a renovação continua manual (P2P). A extensão da assinatura é escrita à mão em `customers/{uid}/subscriptions` até ao Blaze.

## Versão 0.188 — Pagamento da renovação por Revolut

- `PAGAMENTO_P2P.revolut` = `https://revolut.me/rodrig0almeida`. A folha de renovação mostra um botão azul
  "Pagar com Revolut · 10 €" que abre o link numa nova janela.
- MB WAY, IBAN e titular só aparecem quando estiverem preenchidos (valores que não começam por "(").
- Traduzido em pt, en e fr.
- Testado: botão com o link da Revolut, MB WAY e IBAN escondidos enquanto são marcadores, texto em inglês.
- Nota: o link `revolut.me` não permite definir o valor. O utilizador tem de escrever os 10 € no ecrã da Revolut.

## Versão 0.189 — Prazo de ativação na renovação

- A folha de renovação indica que a ativação pode levar entre 1 hora e 24 horas após a receção do pagamento.
- Traduzido em pt, en e fr.
- Testado: o prazo aparece em português e em inglês.
- O texto sobre cancelamento e devolução de valores não está no código. Se for para a folha de renovação, deve ser enviado
  para ser incluído.

## Versão 0.190 — Informação de cobrança na renovação

- A folha de renovação informa que será cobrado o valor referente ao período de utilização: "Será cobrado o valor
  referente ao período de utilização: 10 € (1 mês)".
- Traduzido em pt, en e fr. Testado em português e em inglês.

## Versões 0.191 e 0.192 — Condições de renovação

- 0.191: folha "Condições de renovação", com ligação a partir da renovação. Contém compra pontual, ativação, direito de desistência (14 dias, DL 24/2014), início antes do fim do prazo, reembolso, cancelamento e reclamações. Traduzida em pt, en e fr.
- 0.192: o cálculo proporcional do início antecipado tem exemplo: após 7 dias de um período de 30, paga 7/30 de 10 €, ou seja, 2,33 €.
- O texto é informativo e deve ser validado por jurista ou pela Direção-Geral do Consumidor antes de publicar.

## Versão 0.194 — Compra de acesso para quem não tem assinatura

- Botão "Comprar acesso" no cartão de Assinatura (quando não há assinatura), no aviso do modo básico e na folha de limite.
  Abre a folha de renovação com os dados de pagamento.
- O botão de pagamento diz só "Pagar · 10 €". Indica-se que qualquer cartão ou banco serve, não só a Revolut.
- Testado: o botão aparece sem assinatura, a nota do banco aparece e o texto do botão não menciona a Revolut.

## Versão 0.195 — Avisos no ecrã Você, abaixo da conta

- Os avisos (renovação, só leitura, instalação da app e modo básico) deixam de aparecer no topo de todos os ecrãs.
  Ficam agora no ecrã Você, logo abaixo do cartão da conta (contentor `avisosConta`, função `contaAvisos()`).
- Consequência: fora do ecrã Você estes avisos deixam de aparecer. Se for preciso, um aviso curto pode voltar noutros ecrãs.
- Testado: o aviso de renovação fica no contentor, logo abaixo do cartão da conta, e não no topo da app.

## Versão 0.196

- Retirada a frase "Não é preciso ter conta Revolut." da nota de pagamento (pt, en, fr). Fica só "Pode pagar com qualquer cartão ou banco."

## Versão 0.197

- Texto de pagamento na renovação reescrito como texto corrido: forma de pagamento, envio do comprovativo, valor cobrado, prazo de ativação. Traduzido em pt, en e fr.
- Removida a nota repetida "Pode pagar com qualquer cartão ou banco." por cima do botão.

## Versão 0.198

- A chave `m_voltar` não existia nos dicionários, e o botão Voltar da folha de condições mostrava o nome técnico. Definida em pt (Voltar), en (Back) e fr (Retour).

## Versão 1.0.0 — Atualizações betas (Modo Dev)

- Versão de lançamento: `APP_VERSION = '1.0.0'`.
- Modo Dev → Atualização → "Atualizações betas". Só neste aparelho (`localSettings.betaUpdates`).
- Com a opção ligada, a app usa `indexb.html` no mesmo servidor. O `index.html` continua a ser a versão estável.
  - Ao abrir `index.html` com a opção ligada, a app passa para `indexb.html`.
  - Ao abrir `indexb.html` com a opção desligada, passa para `index.html`.
  - A função `redirecionarBeta()` faz esta verificação no primeiro carregamento. Cada página só redireciona se a opção não bater com a página atual, por isso não há ciclos.
- Para testar uma versão: publica o ficheiro novo como `indexb.html`. Os aparelhos com a opção ligada atualizam-se pela verificação normal de versão.
- Atenção: se `indexb.html` tiver um erro grave, a opção fica ligada e a app não volta sozinha ao `index.html`. Para sair, é preciso abrir a app com a opção desligada ou apagar `localSettings` no armazenamento do site.

## Versão 1.0.1 — Chaves do armazenamento com prefixo "corrida:"

- O domínio github.io é partilhado com outras páginas, por isso todas as chaves do localStorage passam a ter o prefixo `corrida:` (ex.: `corrida:bills`, `corrida:localSettings`). A constante `CHAVE_PREFIXO` aplica-o no armazenamento de fallback.
- As leituras diretas (no `<head>` e em `LANG`) usam já a chave prefixada.
- Migração única (no `<head>`, antes de qualquer leitura): copia para `corrida:<chave>` os valores antigos sem prefixo que passam uma validação de formato. Não apaga nada. Marca a migração com `corrida:ns`.
- Só é copiado o que passa a validação. O `profile` de outra página, por exemplo, não é copiado se não tiver `name` ou `photo`.
- Chaves citadas no pedido que não existem no código: `pendingCloudPush`, `appSettings`. Não foram criadas.
- Testado: dados válidos copiados, `profile` de outra página fora da migração, chaves antigas mantidas, app a ler os dados migrados.
- A nuvem não é afetada. Se algum valor local ficar errado, a sincronização repõe-no.

## Versão 1.0.2 — Modo demonstração (Modo Dev)

- Modo Dev → "Modo demonstração". Guardado neste aparelho (`corrida:demoMode`). Ativar ou desativar recarrega a página.
- Usa um armazenamento separado (`corrida-demo:`). A constante `CHAVE_PREFIXO` escolhe o prefixo ao arrancar. Os dados reais não são lidos nem alterados.
- Começa limpo. Conta de demonstração (`DEMO_USER`) e assinatura ativa de demonstração (`demoSubscription()`), para não aparecerem avisos de limite nem de renovação.
- Nada vai para a nuvem: `pushToFirestore`, `pushToCloud`, `scheduleCloudPush` e `_doCloudSync` saem logo em demonstração. A sincronização do Firestore não é ligada.
- "Limpar dados da demonstração" apaga só as chaves `corrida-demo:*`.
- Aviso "Modo demonstração" no ecrã Você. Sair da conta de demonstração desativa o modo.
- Testado: utilizador e assinatura de demonstração, perfil gravado só no armazenamento de demonstração, dados reais intactos, sem gravação na nuvem, regresso aos dados reais ao desativar.

## Versão 1.0.3 — Correção do ciclo com demonstração e betas

- Problema: com o modo demonstração ativo, a opção de atualizações betas era lida da área de demonstração (`corrida-demo:`), onde não existe. Isso fazia o redirecionamento comportar-se de forma errada e a tela de carregamento ficava a repetir.
- Correção: a opção de betas é do aparelho. `betaLigada()` e `setBetaLigada()` leem e gravam sempre `corrida:localSettings`.
- Proteção contra ciclos: `redirecionarBeta()` não redireciona outra vez nos 20 segundos seguintes (`sessionStorage` `corrida:redir`).
- Testado num servidor local com `index.html` e `indexb.html`: com betas ligadas, a sequência é index → indexb, uma vez. Com demonstração ativa, indexb não volta a redirecionar.

## Versão 1.0.4

- Email da conta de demonstração: `corridaplus@corridaplus.app` (antes `demo@corridaplus.app`).

## Versão 1.0.5

- Removido o aviso "Modo demonstração" no ecrã Você e o rótulo "Conta de demonstração". A conta de demonstração aparece como uma conta normal, para captura de ecrãs. O interruptor continua no Modo Dev.

## Versão 1.0.6

- `apple-mobile-web-app-status-bar-style` passa de `black-translucent` para `black`. A barra de estado do iPhone fica sólida, sem mostrar a página ou o ecrã de arranque por baixo nos primeiros segundos. O conteúdo começa abaixo da barra (a app já usa `env(safe-area-inset-top)`).
- Com `black`, a barra fica preta no iPhone e não segue a cor do tema, porque o iOS ignora `theme-color` nesse modo.

## Versão 1.0.7 — Arranque em fases

1. **Dados locais primeiro.** A app lê e desenha o conteúdo do armazenamento local sem esperar pela rede.
2. **Atualização limitada a 1,2 s.** `checkForUpdatesOnStartup(1200)` corre antes de mostrar o conteúdo. Se houver versão nova, recarrega. Sem resposta a tempo, a app abre normalmente. Sem sessão, mostra o alerta depois de abrir. A decisão usa `corrida:contaLigada` (sinal local), porque a conta ainda não é conhecida nessa fase.
3. **Conta e Firebase em segundo plano.** `arrancarConta()` espera até 5 s pela confirmação da conta, sem bloquear a abertura. Com conta, `ligarConta()` liga os listeners e a primeira sincronização. A pastilha mostra "sincronizando…" (`setSyncBusy()`) e depois "sincronizado" ou "offline".
4. **Ecrã de carregamento mais curto.** Tempo mínimo 0,4 s (antes 1,2 s). Pausa antes do desvanecimento 0,35 s (antes 0,9 s).
- A app arranca logo (`arrancarApp()`), sem esperar pelo evento `firebase-ready` nem pelo limite de 3 s. O Firebase é esperado só dentro de `arrancarConta()`.
- Testado num servidor local sem Firebase: o ecrã de carregamento sai aos 0,9 s, com os dados locais desenhados e sem erros.

## Versão 1.0.8 — Abertura mais rápida

- **SheetJS fora do arranque.** Deixou de ser um `<script>` no `<head>` (cerca de 900 KB, que bloqueava a primeira pintura). Passa a ser carregada por `carregarXlsx()` só quando o utilizador exporta para Excel.
- **Verificação de versão sem descarregar a página inteira.** Há um `<meta name="app-version">` no início do `<head>`. `lerVersaoRemota()` pede só os primeiros 4 KB (`Range: bytes=0-4095`) e lê essa meta. Se a meta não existir, lê a página completa como antes.
  - Atenção: a meta `app-version` e `const APP_VERSION` têm de ser atualizadas sempre juntas.
  - O ganho depende de o servidor responder a pedidos `Range`. Se não responder, a leitura continua correta, só não fica mais rápida.
- Testado: a versão remota é lida (1.0.8), a biblioteca não é pedida no arranque e o ecrã de carregamento sai aos ~1 s, sem erros.
- Ainda no caminho de arranque: o tempo mínimo do ecrã de carregamento (0,4 s + 0,35 s + 0,4 s de desvanecimento) e a verificação de atualização (até 1,2 s) na primeira abertura de cada sessão.

## Versão 1.0.2a — Correção do ciclo ao atualizar

- Causa: a app lia a versão publicada pela meta `app-version` do início do ficheiro. Se `APP_VERSION` e essa meta não coincidirem, o aparelho vê sempre uma versão diferente, recarrega e repete.
- Correção 1: `podeAtualizarPara(versao)` (sessionStorage `corrida:updTentativa`). A mesma versão remota não é tentada duas vezes seguidas na mesma aba. Aplica-se à atualização automática e à manual.
- Correção 2: `APP_VERSION` e a meta `app-version` estão ambas em `1.0.2a`. Ao mudar a versão, muda sempre as duas.
- Versão 1.0.2a é menor que 1.0.8. Como a verificação é "diferente de", o aparelho com 1.0.8 passa para 1.0.2a (é um retrocesso, aceite de propósito).
- Testado: (A) aparelho 1.0.2a com servidor 1.0.8: atualiza uma vez. (B) meta 1.0.8 com versão 1.0.2a: recarrega uma vez e para, sem ciclo.
