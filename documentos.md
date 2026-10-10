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

> **Nota (v0.4.11):** a união por ID e as lápides descritas nas secções 3 e
> 3.1 foram substituídas pela fusão com base em três vias (ver a entrada da
> versão 0.4.11 na secção 10). A secção 3 fica como histórico.

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

## 10. Alterações desta versão (até v0.74)
- **Versão 0.74 — seletor de tema desliza ao clicar**: a mudança suave de cores (0.50) aplicava
  uma transição de cores a todos os elementos, incluindo a gota. Por isso, ao clicar numa opção
  de tema (Claro · Sistema · Escuro), a gota saltava em vez de deslizar. Agora a gota mantém a
  sua animação de deslize durante a mudança de tema. Arrastar já funcionava.

- **Versão 0.73 — cantos concêntricos também no modo transparente**: a regra estava definida,
  mas no modo transparente uma regra anterior fixava as gotas em 14px, mais do que o contentor
  (12px) permite. Por isso a gota ficava fora de alinhamento com a moldura. Agora as gotas e as
  opções dos seletores, e as abas da barra, têm prioridade e seguem a regra nos dois modos.

- **Versão 0.72 — rótulo "Padrão" alinhado ao ímã**: os rótulos Transparente / Padrão / Opaco
  estavam distribuídos com espaço entre eles, e o do meio ficava fora do ponto 50%. Agora o
  "Padrão" fica centrado exatamente na posição 50%, onde o cursor encaixa.

- **Versão 0.71 — balões sempre tintados; interruptor da imagem de fundo**:
  - Removida a opção "Balões tintados". Os balões ficam sempre com a cor do tema.
  - Em Configurações → Aparência, a imagem de fundo tem um interruptor de ativar ou desativar.
    Desligada, a imagem fica guardada e volta a aparecer ao ligar. Por omissão, fica ativa
    quando existe imagem (`localSettings.bgEnabled`). O modo claro com texto claro sobre o
    fundo também só se aplica com a imagem ativa.

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

- **Versão 0.69 — correção: temas em tempo real**: a função `softChange()` tinha sido apagada
  por engano numa alteração anterior. Por isso, ao mudar a cor do tema, o modo claro/escuro
  ou o efeito transparente, a mudança falhava e a animação gradual não corria. Reposta.
  Confirmado que não falta mais nenhuma função de código.

- **Versão 0.68 — "ímã" na intensidade do efeito transparente**: a barra de intensidade encaixa
  em 0 (transparente mais), 50 (padrão) e 100 (opaco), com uma margem de 6 pontos. Perto
  desses valores, o cursor vai para a posição exata. Os valores intermédios continuam
  disponíveis, longe dos três pontos.

- **Versão 0.67 — barra de abas fixa (Android e iOS) e efeito transparente sem reiniciar**:
  - **Causa**: a página inteira rolava. Ao rolar, a barra de endereço do navegador recolhe ou
    expande, e a barra de abas, que é fixa, saltava de lugar. Acontecia em páginas mais compridas
    e no Android. A camada `translateZ` da 0.64 foi removida, porque não resolvia.
  - **Correção**: só a área das telas (`#app`) rola. O corpo da página fica fixo, e a barra de
    abas nunca se mexe. Ao trocar de aba, a área volta ao topo.
  - **Efeito transparente**: o fade ao ligar ou desligar termina sempre (`finally`), mesmo que
    algo falhe. Assim a interface não fica a meio fade e não precisa de reiniciar.

- **Versão 0.66 — cantos concêntricos com a borda contada**: os valores passam a contar a
  borda, além do espaçamento. Seletores: 12 − (1,5 + 3) = 7,5px nas opções e na gota. Barra
  de abas: 20 − (1 + 7) = 12px nas abas e na gota. Computador (barra lateral): 20 − (1 + 12) = 7px.
- **Versão 0.65 — cantos aninhados concêntricos**: aplicada a regra "raio interno = raio externo
  − espaçamento". Seletores (Dia·Semana·Mês·Ano, tema, frequência, turnos): contentor 12px,
  espaçamento 3px, opções e gota (valores finais na 0.66). Barra de abas: contentor 20px. Os círculos do seletor de cor
  e os interruptores não precisam de ajuste.
- **Versão 0.64 — barra e fundo estáveis ao trocar de aba (diagnóstico, sem iPhone)**:
  ao trocar de aba, a app chamava sempre `window.scrollTo(0,0)`. No iOS, isso faz a barra de
  endereço recolher ou expandir, e as camadas fixas (barra, fundo e camada de vidro) podem
  saltar ou parecer redimensionadas. Agora só volta ao topo se a página estiver rolada, e
  as camadas fixas ficam em composição própria (`translateZ(0)`), para não serem repintadas
  na troca de aba. Não foi possível reproduzir: confirmar no iPhone.
- **Versão 0.63 — removido o deslize da página**: o deslize na horizontal sobre o conteúdo
  que mudava de aba (das versões 0.61 e 0.62) foi removido. Mantém-se a animação de deslize
  quando se toca numa aba (a nova entra pelo lado de onde vem). Continua a funcionar o arrasto
  da gota na barra de abas.
- **Versão 0.62 — barra de abas volta a arrastar a gota**: a alteração da 0.61 que impedia o
  arrasto na barra estava errada. Ao deslizar sobre a barra, a gota volta a seguir o dedo,
  e ao soltar muda para a aba mais próxima, como antes.
- **Versão 0.61 — deslizar na página segue o dedo**:
  - **Página** (só telemóvel): ao deslizar na horizontal, a aba atual acompanha o dedo e
    a vizinha aparece do lado. Ao soltar, muda se o deslize passar 25% da largura ou for
    rápido; senão volta. A vizinha é a da ordem da barra.
  - Não atua sobre campos, botões, seletores, interruptores nem gráficos, nem com uma
    folha aberta. Em ecrãs largos (computador), o deslize não está ativo.
  - Limitação: não testado num iPhone. A sensação de seguir o dedo depende do aparelho.

- **Versão 0.60 — troca de abas com deslize, sem fade**: desde a 0.56 e anteriores, as
  telas tinham uma animação de opacidade (0 a 100%), e os cartões pareciam transparentes
  a ficar opacos, como se a página estivesse a carregar. Agora:
  - A nova aba entra com um deslize de 28 px, do lado de onde vem (direita ao avançar,
    esquerda ao voltar), com opacidade total desde o início.
  - A ordem é a das abas da barra (`SCREEN_ORDER`). Trocar pelos gestos ou pela barra
    usa a mesma animação.
  - Limitação: a aba que sai não faz deslize de saída; desaparece de imediato.

- **Versão 0.59 — ajustes da imagem de fundo**: só aparecem quando há uma imagem.
  - **Véu sobre a imagem** (0 a 100%): 0% é o véu leve atual; 100% é o véu totalmente
    opaco, que tapa a imagem.
  - **Desfoque com granulação** (0 a 100%): aplica desfoque (até 20 px) e uma camada de
    ruído com modo de mistura "sobreposição", que cresce com o desfoque.
  - Os dois ajustes atualizam ao vivo, sem fade. Só trocar de imagem tem fade.
  - Guardado só neste aparelho (`localSettings.bgVeil` e `localSettings.bgBlur`).
  - Limitação: o desfoque em ecrãs de telemóvel pode pesar. Se a app ficar lenta com
    o desfoque no máximo, reduzo o valor máximo.

- **Versão 0.58 — intensidade do efeito transparente e balões tintados**: com o efeito
  ligado, Configurações → Aparência mostra uma barra de **intensidade** (0 a 100, padrão
  50). Os valores intermédios são possíveis. O rótulo muda entre "Mais transparente",
  "Padrão" e "Mais opaco". Escala a transparência e o desfoque das barras, cartões,
  folhas e botões (`--gl-k` e `--gl-b`). Com o padrão (50), o aspeto é o atual.
  - **Balões tintados** (ligado por omissão): a gota e os seletores usam a cor do tema.
    Desligado: neutros e sem cor.
  - Guardado só neste aparelho (`localSettings.glassLevel` e `localSettings.glassTint`).

- **Versão 0.57 — sem efeito de subida ao desligar o efeito transparente**: a animação
  das telas no modo normal deslizava 6 px de baixo para cima. Ao desligar o efeito, o
  nome da animação mudava e ela recomeçava, parecendo que a página subia. Agora a
  animação das telas é só de opacidade, nos dois modos. A troca de abas no modo normal
  deixa de ter o pequeno deslize.

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

- **Versão 0.55 — ligar/desligar o efeito transparente sem piscar**: a função do
  interruptor redesenhava todo o ecrã de Configurações (`renderSettings()`), e a imagem
  de fundo repetia o fade. Agora só o interruptor é atualizado, e a imagem não é
  reanimada. Assim, o efeito muda como a troca de tema, que já não piscava.

- **Versão 0.54 — legibilidade com imagem de fundo no modo claro**: ao escolher a imagem,
  a app calcula o tom médio (luminosidade, já com o véu). Se for escura, no modo claro
  o texto que está diretamente sobre o fundo (títulos, datas dos seletores, mensagens
  vazias, linhas de despesas, botão "Adicionar") fica claro. Os cartões brancos e os
  campos mantêm o texto escuro. Imagens antigas são analisadas na primeira abertura.
  O modo escuro não muda.
  - Limitação: a média pode não bater em imagens com partes muito escuras e muito
    claras ao mesmo tempo. Nesse caso, o texto pode continuar pouco legível numa das zonas.

- **Versão 0.53 — imagem de fundo sem esbranquiçar no modo claro**: o véu no modo claro
  era creme a 55%, o que lavava a imagem. Agora é um véu escuro leve (22%), que mantém
  as cores da imagem e continua legível. O modo escuro não muda (véu escuro a 62%).

- **Versão 0.52 — imagem de fundo com fade**: a camada da imagem (`#customBg`) deixou
  de estar oculta com `display:none`, o que impedia qualquer transição. Agora:
  - Ao ativar ou desativar o efeito transparente, a imagem entra de novo com fade.
  - Ao trocar de imagem ou remover, a atual sai (fade), a nova entra.
  - Ao mudar o modo claro/escuro, a imagem troca o véu com a mesma transição.

- **Versão 0.51 — seletor de cor do tema com animação**: escolher uma cor já não
  redesenha a grade. Só a seleção muda nos círculos existentes, com mola e um pequeno
  encolhimento ao tocar.
- **Versão 0.50 — mudanças suaves**: trocar a cor do tema, o modo claro/escuro ou o
  efeito transparente faz as cores passar gradualmente (cerca de meio segundo). A
  camada de vidro aparece ou desaparece com opacidade. A imagem de fundo personalizada
  continua a mudar de uma vez.
- **Versão 0.49 — animações também no modo normal**: a gota com deslize, deformação e
  mola funciona nos dois modos. No modo normal é sólida, com a cor do tema, e sem
  desfoque. O deslize sobre o conteúdo também muda de aba.
- **Versão 0.48 — deformação da gota também na vertical**: ao deslizar, estica até 20%
  no sentido do movimento e comprime até 18% na altura. Ao chegar, começa a 116% na
  largura e 82% na altura, e recupera com mola.
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

- **Versão 0.46 — arrastar a gota no celular**: no telemóvel, o navegador tomava o
  movimento horizontal como deslizar da página e cancelava o arrasto. Ao cancelar, a
  app usava coordenadas inválidas e voltava sempre à primeira aba. Agora a barra
  fica com o toque para ela (`touch-action: none`, só com o efeito transparente),
  a navegação usa a última posição real do dedo, e um cancelamento só devolve a gota
  à aba atual, sem mudar de aba.

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

- **Versão 0.4.44 — interruptor desligado visível sem o efeito transparente**: o trilho
  desligado usava uma cor muito translúcida (18%), que no modo escuro quase
  desaparecia. Agora é mais opaco no modo claro (24%) e tem uma cor própria no modo
  escuro normal. As regras do efeito transparente continuam a aplicar-se só com o
  efeito ativo.

- **Versão 0.4.43 — interruptor ligado com a cor do tema nos dois modos**: o interruptor
  ligado usa agora `--switch-on`, definida ao escolher a cor do tema. No modo
  transparente escuro, a regra do interruptor desligado sobrepunha o estado ligado,
  por isso ficava translúcido. Agora a regra só se aplica aos interruptores desligados.

- **Versão 0.4.42 — separador e imagem de fundo**:
  - Um separador visível (linha) divide o bloco Aparência do bloco Configurações.
  - Em Configurações → Aparência, "Imagem de fundo": escolher uma imagem (reduzida para
    no máximo 1080 px, sem cortar) e remover. Fica guardada só neste aparelho
    (`localSettings.bgImage`) e aparece atrás de toda a app, com um véu para manter a
    legibilidade (claro ou escuro, conforme o modo).
  - Redefinir as configurações também remove a imagem de fundo.

- **Versão 0.4.41 — opção ativa do seletor em verde**: com o efeito transparente, a
  opção selecionada de Dia · Semana · Mês · Ano voltou a ter o fundo da cor do tema
  (`--vault`), com texto branco, para indicar claramente que está ativa.

- **Versão 0.4.40 — efeito transparente em Configurações e nos controlos**:
  - O interruptor "Efeito transparente (liquid glass)" sai do Modo Dev e passa para
    Configurações → Aparência. Continua a ser uma preferência deste aparelho.
  - Animação das abas: no modo transparente, a troca de abas só anima a opacidade.
    O deslocamento vertical anterior fazia o vidro tremer.
  - Com o modo ativo, também ficam em vidro: a aba selecionada, os seletores
    Dia · Semana · Mês · Ano (com a opção ativa em vidro), as setas de navegação e
    os interruptores.

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

- **Versão 0.4.38 — seleção da cor do tema visível no modo claro**: a cor escolhida
  tinha a borda na cor de texto, que no modo claro é escura e se confundia com a
  própria cor verde. Agora a seleção é um anel com um espaço claro à volta, visível
  em qualquer tema.

- **Versão 0.4.37 — suporte com log em ficheiro**: o texto do email passa a dizer
  "Mensagem:" (sem a palavra "problema"). O log das últimas 24 horas segue como
  ficheiro .txt (`corridaplus-log-AAAA-MM-DD-HHMM.txt`).
  - Se o aparelho permitir partilhar ficheiros (em geral, telemóvel), abre a folha de
    partilha com o ficheiro anexado e o texto do email. Esta folha não preenche o
    destinatário; o endereço aparece no texto.
  - Caso contrário, o ficheiro é transferido e o email abre com o endereço preenchido,
    pedindo para anexar o ficheiro. Um email não consegue anexar ficheiros por si.

- **Versão 0.4.36 — botão de suporte**: no fim de Configurações, abaixo da zona de
  perigo, o botão "✉️ Suporte" abre o email com o endereço de suporte já preenchido.
  O assunto e o corpo incluem o email e o ID da conta (ou "sem conta Google ligada"),
  e a versão da app. O botão funciona também em modo básico.

- **Versão 0.4.35 — alerta de atualização sem sessão**: sem sessão Google, a app
  deixa de recarregar sozinha quando há uma versão nova. Mostra uma folha com a
  versão nova e três opções: "Atualizar agora", "Entrar com Google" (para receber
  as atualizações automaticamente) e "Mais tarde". Com sessão, a atualização
  automática continua igual. O alerta só aparece depois das boas-vindas.

- **Versão 0.4.34 — cópia diária na planilha**: com sessão e endereço da planilha
  configurado, ao abrir a app é enviada uma cópia do estado atual para a planilha,
  no máximo uma vez por dia (`localSettings.lastSheetBackup`, device-local). É só
  um envio: não lê nem funde dados da planilha. A cópia respeita a encriptação, se
  estiver ativa. Não corre enquanto houver uma decisão pendente sobre os dados. O
  resultado fica no log de sincronização (Modo Dev).
  - Limitação: a cópia só acontece quando a app é aberta. Se a app não for aberta
    num dia, a cópia desse dia não acontece.

- **Versão 0.4.33 — endereço da planilha na conta e apagado com tudo**:
  - O endereço da planilha de sincronização passa a ser guardado em `profile.syncUrl`,
    no perfil da conta Google. Assim, segue a conta entre aparelhos. Sem sessão, não
    é guardado nem usado. Um endereço que já existisse neste aparelho passa para a
    conta no primeiro login.
  - **Apagar tudo** apaga também o endereço da planilha, neste aparelho e na conta.
    Redefinir só as configurações e apagar só os dados continuam a manter o endereço.
  - Nota: o endereço é uma credencial da planilha. Fica no documento da conta, que as
    regras do Firestore só deixam ler e escrever ao próprio utilizador.

- **Versão 0.4.32 — modo básico sem planilha**: sem sessão Google, a app não lê nem
  envia para a planilha (Apps Script). Antes, com um URL guardado, a primeira
  sincronização perguntava se queria os dados da nuvem, mesmo sem conta. Agora
  `_doCloudSync()` e `pushToCloud()` saem logo quando não há sessão, e
  `scheduleCloudPush()` também. A planilha continua a funcionar para quem tem login.

- **Versão 0.4.31 — tela de carregamento sem conta**: sem sessão Google, mas com um
  URL de sincronização guardado, a app esperava pela sincronização antes de esconder
  a tela de carregamento. A ligação podia demorar até 15 s por tentativa, e a tela
  ficava presa. Agora a sincronização corre em segundo plano, como já acontece com
  a conta Google.

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

- **Versão 0.4.28 — cores do modo escuro seguem o tema**: o separador ativo da barra
  (e a opção selecionada nas plataformas e no seletor de mês) usava um amarelo fixo
  em modo escuro. Agora usa a cor de texto do tema escolhido, com um fundo neutro.

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

- **Versão 0.4.26 — reposição volta ao padrão, incluindo turnos**:
  - Ao apagar configurações ou tudo, a base de sincronização passa a ser o estado
    já reposto. Antes ficava vazia, e a versão antiga da nuvem era tratada como
    primeira sincronização, o que restaurava as configurações antigas (turnos
    incluídos).
  - Durante a reposição, as alterações vindas da nuvem são ignoradas
    (`window._suppressRemote`), até o novo estado ser gravado.

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

- **Versão 0.4.21 — versão no fim de Configurações**: o rodapé "Corrida+ vX.X.X"
  passou para o fim do ecrã, abaixo da zona de perigo e do Modo Dev. Continua a
  abrir o Modo Dev com 5 toques.

- **Versão 0.4.20 — ajustes de ganhos, detalhamento e turnos**:
  - **Ganhos**: removida a linha "líq. €" por baixo do valor de cada ganho.
  - **Lucro · Detalhamento**: o tracejado que ficava por baixo de "Comissão
    plataforma" foi removido. Fica só o tracejado por baixo de "Líquido TVDE".
  - **Turnos com nome configurável**: em Configurações → Turno de trabalho, quando o
    interruptor está ligado, aparecem quatro campos (um por turno), com máximo de 14
    letras. Em branco, usa o nome original. Os nomes são guardados em
    `profile.turnoNames`, que sincroniza entre aparelhos, e aplicam-se ao formulário de
    ganho, ao gráfico "Ganhos por turno" e à exportação para Excel.

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

- **Versão 0.4.13 — Configurações agrupadas**: as secções "Registos opcionais",
  "Meta de lucro" e "Outras plataformas" passaram a ser um só grupo, "Opcionais",
  com os interruptores de turno, km, IVA/imposto, metas de lucro e outras
  plataformas. A comissão continua num grupo próprio, logo abaixo.
- **IVA / Imposto com chave de ativação** (`profile.ivaEnabled`). Desligado, `ivaRate()`
  devolve 0 e todos os cálculos e ecrãs que dependem do IVA deixam de o mostrar. A
  taxa continua guardada, para voltar a ativar sem a introduzir de novo. Por
  omissão, fica ativo quando já existe uma taxa maior que 0.

- **Versão 0.4.12 — reversão**: o gráfico diário de Lucro (Semana) volta à fórmula
  anterior, `(bruto − combustível − IVA sobre o bruto sem combustível) + pessoal −
  outras variáveis`, e à legenda "Lucro líquido de cada dia da semana". A
  alteração da 0.4.10 (item 14) foi revertida a pedido.

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

## 18. Login Google voltou a ser opcional, por enquanto (v4.0.1)

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

## 19. Esquema de versão voltou a pré-1.0 (v0.4.2)

A numeração 4.x.x usada temporariamente durante a integração Firebase foi
abandonada. De volta a 0.x.x — a v1.0.0 fica reservada para o lançamento
com o recurso completo de assinatura paga (ver `corridaplus-firebase-plan.md`,
Fase 3).

## 20. Turno de trabalho (opcional) + gráfico "Ganhos por turno" (v0.4.2)

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

## 21. Plataformas personalizadas (opcional) (v0.4.2)

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

## 22. Pacote de ajustes (v0.4.3)

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

## 23. Reforço de segurança: App Check + regras do Firestore com validação de schema (v0.4.4)

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

## 24. Cartão "Assinatura" em Configurações, com data de validade (v0.4.5)

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
