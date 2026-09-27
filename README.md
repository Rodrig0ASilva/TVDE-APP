# 🚖 Corrida+ (Gestão Financeira para Motoristas TVDE)

O **Corrida+** é um aplicativo web focado na gestão financeira de motoristas de aplicativo (Uber, Bolt e corridas particulares). Projetado com foco na experiência mobile, ele funciona como uma SPA (*Single Page Application*) com visual nativo de aplicativo (estilo iOS). 

O grande diferencial do projeto é sua arquitetura híbrida: ele roda de forma 100% instantânea salvando os dados no dispositivo através do `localStorage` e utiliza **Google Drive** como banco de dados em nuvem ¨gratuito¨ e sem servidor (Serverless), permitindo sincronizar os seus dados entre vários aparelhos (ex: celular e tablet) em tempo real.

---

## ✨ Funcionalidades Implementadas

### 🚗 Ganhos por Plataforma
* **Registro Multi-Plataforma:** Lance os ganhos separadamente por Uber, Bolt, corridas Particulares e Outros (entregas, bicos extras, etc).
* **Visão Diária, Semanal e Mensal:** Acompanhe o faturamento bruto em qualquer período, com navegação rápida entre datas.
* **Total Bruto Consolidado:** Veja de imediato o somatório de tudo o que entrou (TVDE + Pessoal) num único card em destaque.

### 💸 Despesas Fixas e Variáveis
* **Despesas Fixas Recorrentes:** Cadastre contas mensais (aluguel, financiamento, internet, etc.) que se repetem automaticamente mês a mês.
* **Despesas Variáveis:** Lance combustível, manutenção, alimentação e outras despesas do dia a dia, por categoria.
* **Dados Ocultos e Limpeza:** No Modo Dev, é possível visualizar e excluir despesas fixas ocultadas ou registros antigos sem perder o histórico dos demais meses.

### 🧮 Cálculo Automático de IVA e Comissão
* **IVA Configurável:** Defina a sua taxa de IVA (%) nas configurações — o app desconta automaticamente do faturamento bruto das apps (Uber/Bolt).
* **Comissão da Plataforma:** Defina também a taxa de comissão cobrada pela própria Uber/Bolt, descontada logo após o IVA.
* **Detalhamento Completo:** A aba Lucro mostra o passo a passo exato: Bruto → IVA → Comissão → Líquido TVDE → + Particular/Outros → − Combustível → − Despesas fixas → − Outras variáveis → **Lucro líquido final**.

### 📊 Estatísticas e Comparativos
* **Gráficos Ganhos vs. Despesas:** Visualize por dia, semana ou mês, com barras lado a lado para comparar entrada e saída de dinheiro.
* **Separação TVDE vs. Pessoal:** Estatísticas mostram claramente o que veio de Uber/Bolt (sujeito a IVA e comissão) e o que veio de corridas particulares (sem desconto).
* **Evolução dos Últimos Meses:** Acompanhe a tendência de lucro ao longo do tempo.

### 🛠️ Modo Dev (Avançado)
* **Acesso Oculto:** Toque 5 vezes no número da versão, nas configurações, para ativar o Modo Dev.
* **Dados Ocultos e Limpeza:** Visualize e exclua registros (ganhos, despesas, contas fixas ocultadas) de qualquer mês.
* **Importar Dados:** Cole um JSON de backup para restaurar seus dados manualmente, caso a sincronização automática falhe.

---

## 🚀 Como Configurar o Banco de Dados (Google Drive)

Para sincronizar os seus dados entre aparelhos e garantir a segurança com backups, precisamos criar o script no Google Drive e **gerar o link de sincronização**. Siga o passo a passo com atenção:

### Passo 1: Criar o Script e Inserir o Código

1. Acesse o [App Script](https://script.google.com/) e crie um **Novo projeto** em branco (ex: `BD_CorridaPlus`).
2. Apague todo o código que estiver na tela e cole este bloco abaixo:

```javascript
const FILE_NAME = "tvde_dados_sync.json"; // Nome do ficheiro da app TVDE
const FOLDER_NAME = "cofre"; // Pasta principal
const BACKUP_FOLDER_NAME = "cofre/cofre_backups"; // Caminho para a subpasta de backups

// ==========================================
// 1. Função Auxiliar (Lida com subpastas corretamente)
// ==========================================
function getOrCreateFolder(folderPath) {
  const parts = folderPath.split('/');
  let currentFolder = DriveApp.getRootFolder();
  
  for (let i = 0; i < parts.length; i++) {
    let folderName = parts[i];
    let folders = currentFolder.getFoldersByName(folderName);
    
    if (folders.hasNext()) {
      currentFolder = folders.next(); // Entra na pasta se ela existir
    } else {
      currentFolder = currentFolder.createFolder(folderName); // Cria a pasta se não existir
    }
  }
  return currentFolder;
}

// ==========================================
// 2. Função POST (Salvar dados)
// ==========================================
function doPost(e) {
  try {
    const data = e.postData.contents;
    const folder = getOrCreateFolder(FOLDER_NAME);
    let files = folder.getFilesByName(FILE_NAME);
    let file;
    
    if (files.hasNext()) {
      file = files.next();
      file.setContent(data);
    } else {
      file = folder.createFile(FILE_NAME, data, MimeType.PLAIN_TEXT);
    }
    
    return ContentService.createTextOutput(JSON.stringify({status: "success"}))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch(err) {
    return ContentService.createTextOutput(JSON.stringify({error: err.message}))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// ==========================================
// 3. Função GET (Ler dados)
// ==========================================
function doGet(e) {
  try {
    const folder = getOrCreateFolder(FOLDER_NAME);
    let files = folder.getFilesByName(FILE_NAME);
    
    if (files.hasNext()) {
      let file = files.next();
      let content = file.getBlob().getDataAsString();
      return ContentService.createTextOutput(content)
        .setMimeType(ContentService.MimeType.JSON);
    } else {
      return ContentService.createTextOutput(JSON.stringify({}))
        .setMimeType(ContentService.MimeType.JSON);
    }
    
  } catch(err) {
    return ContentService.createTextOutput(JSON.stringify({error: err.message}))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// ==========================================
// 4. Função de Backup Diário
// ==========================================
function fazerBackupDiario() {
  const mainFolder = getOrCreateFolder(FOLDER_NAME);
  let files = mainFolder.getFilesByName(FILE_NAME);

  if (files.hasNext()) {
    let originalFile = files.next();
    
    // Pega a data de hoje no formato YYYY-MM-DD
    let dataHoje = Utilities.formatDate(new Date(), Session.getScriptTimeZone(), "yyyy-MM-dd");
    let nomeBackup = "tvde_backup_" + dataHoje + ".json"; 

    // Vai buscar (ou criar) a pasta de destino usando o caminho completo
    const backupFolder = getOrCreateFolder(BACKUP_FOLDER_NAME);
    
    // Verifica se o backup de hoje já existe para evitar cópias duplicadas
    let existingBackups = backupFolder.getFilesByName(nomeBackup);
    if (!existingBackups.hasNext()) {
      // Faz a cópia apenas se não existir um ficheiro com o nome de hoje
      originalFile.makeCopy(nomeBackup, backupFolder);
    }
  }
}
```

3. Clique no ícone de **Salvar** (disquete) no menu superior.

### Passo 2: Configurar o Backup Automático

Para que o sistema faça cópias de segurança sozinho:
1. No menu lateral esquerdo do Apps Script, clique no ícone de relógio (**Acionadores** ou **Triggers**).
2. Clique no botão azul **Adicionar acionador**.
3. Configure da seguinte forma:
   * **Escolha a função que será executada:** `fazerBackupDiario`
   * **Selecione a origem do evento:** `Baseado no tempo`
   * **Selecione o tipo de acionador com base no tempo:** `Temporizador diário`
   * **Selecione a hora:** Escolha um horário de sua preferência (ex: *Meia-noite a 1h da manhã*).
4. Clique em **Salvar** (se pedir autorização, permita o acesso à sua conta).

### Passo 3: Como Gerar e Pegar o Link (Atenção Aqui!)

Este é o momento de criar a URL que o seu aplicativo vai usar.
1. No canto superior direito do Apps Script, clique no botão azul **Implantar** e escolha **Nova implantação**.
2. Clique no ícone de engrenagem (`⚙️`) ao lado de "Selecione o tipo" e escolha **App da Web**.
3. Preencha os campos **EXATAMENTE** desta forma:
   * **Descrição:** Pode escrever "Versão 1".
   * **Executar como:** *Você (seu-email@gmail.com)*
   * **Quem tem acesso:** Selecione **Qualquer pessoa** *(Se não marcar "Qualquer pessoa", o celular não vai conseguir salvar)*.
4. Clique no botão azul **Implantar**.
5. Vai aparecer um aviso de "Autorização necessária". Clique em **Autorizar acessos** e escolha o seu e-mail.
6. O Google mostrará uma tela dizendo "O Google não verificou este app". Clique na palavra **Avançado** (lá embaixo) e depois clique em **Acessar projeto sem título (não seguro)**. Clique em **Permitir**.
7. Na última tela que aparecer, você verá escrito "URL do app da Web" e um link gigante embaixo (terminando com `/exec`). **Copie este link gigante.**

### Passo 4: Colar o Link no Aplicativo

1. Abra o site do seu aplicativo no seu celular.
2. Navegue até a aba inferior direita chamada **Você** (Configurações).
3. Procure o campo **"URL de Sincronização"** e cole o link gigante lá dentro.
4. Defina a sua taxa de **IVA** e de **Comissão** nas configurações (caso aplicável).
5. Pronto! Faça o mesmo em outro aparelho com o mesmo link para manter os dados sincronizados.

---

## 🛠️ Performance e Sincronização Manual
Para não estourar o limite de acessos diários do Google:
* O app salva localmente e só envia dados para a nuvem alguns segundos após você terminar de fazer uma alteração.
* Ele puxa dados novos automaticamente apenas quando o aplicativo é aberto.
* **Quer puxar atualizações manualmente?** Basta tocar na pílula escrita "sincronizado" ou "offline" no topo do app para forçar a atualização da tela naquele momento.

---

## 📱 Dica: Instale no Celular como um App Real
Abra o site no navegador do celular (Safari ou Chrome), clique no botão de **Compartilhar** do navegador e escolha **"Adicionar à Tela de Início"**. O navegador ocultará as barras de endereço e o ícone ficará junto com seus outros aplicativos.
