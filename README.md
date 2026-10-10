# 🚖 Corrida+

Gestão financeira para motoristas TVDE (Uber, Bolt) e corridas particulares. É uma aplicação web de ficheiro único (`corridaplus.html`), pensada para usar como app no telemóvel, com aspeto nativo (estilo iOS) e suporte ao efeito transparente ("liquid glass").

**Versão atual:** 0.79 (ver a secção 10 da documentação, `corridaplus-docs.md`).

---

## ✨ Funcionalidades

### Ganhos
- Registo por plataforma: **Uber**, **Bolt**, **Particular**, **Outros**, e plataformas extra criadas por ti (ex.: 99, InDrive, Free Now).
- Cada plataforma extra pode cobrar **IVA e comissão** (como Uber/Bolt) ou ser **pessoal** (sem descontos).
- Vistas **Dia, Semana, Mês** e totais por plataforma.
- **Km** (opcional): aparece nas estatísticas, com lucro por km.
- **Turno** (opcional): nomes e emojis personalizáveis, com adição e remoção de turnos. Gráfico "Ganhos por turno".

### Despesas
- **Despesas fixas** diárias, semanais ou mensais. As mensais podem ser repartidas pelas semanas do mês, ou colocadas na primeira semana.
- **Despesas variáveis** por categoria (combustível, manutenção, lavagem, etc.), com editar e apagar.

### Impostos e lucro
- **IVA** e **comissão da plataforma**, cada um com interruptor próprio. Desligados, deixam de entrar nos cálculos.
- Aba **Lucro** com o detalhamento completo: bruto → IVA → comissão → líquido TVDE → pessoal → combustível → despesas → **lucro líquido**.

### Estatísticas
- Vistas **Dia, Semana, Mês e Ano**, com gráficos de ganhos vs. despesas, distribuição por plataforma e por categoria.
- **Metas de lucro** mensal e semanal (opcionais), com barra de progresso.
- **Resumo de ontem** (opcional): ao abrir a app pela primeira vez no dia, mostra o resultado do dia anterior.

### Conta, sincronização e cópias
- **Login com conta Google** para sincronizar os dados entre aparelhos (Firestore, em tempo real).
- **Modo básico** (sem login): até 5 ganhos, 5 despesas variáveis e 5 despesas fixas, com as configurações bloqueadas (exceto IVA e comissão, para testar).
- **Planilha (Google Drive, via Apps Script)** para quem tem login: cópia diária automática ao abrir a app.
- **Exportar dados**: planilha Excel (.xlsx) para a declaração de IRS, ou arquivo de configurações (.json) com dados e configurações.
- **Apagar**: escolhe entre apagar só os dados, redefinir só as configurações, ou apagar tudo (inclui sair da conta). Cada opção pede um código de confirmação.

### Aparência
- **Temas de cor** e **modo claro / escuro / sistema**.
- **Efeito transparente (liquid glass)** com barra de intensidade (0 a 100, com "ímã" nos pontos Transparente, Padrão e Opaco).
- **Imagem de fundo** personalizada, com interruptor, véu e desfoque com granulação.
- Animações de troca de abas e seletores, com gota deslizante e deformação.

### Outros
- **Suporte** por email, com a conta e a versão já preenchidas, e o log das últimas 24 horas em ficheiro.
- **Modo Dev** (5 toques no número da versão): importar dados, ver o log de sincronização, forçar efeitos, e definir o endereço da planilha.

---

## 🛠️ Estrutura

| Ficheiro | Conteúdo |
|---|---|
| `corridaplus.html` | A aplicação completa (HTML, CSS e JavaScript num só ficheiro) |
| `corridaplus-docs.md` | Documentação técnica e histórico de alterações |
| `corridaplus-firebase-plan.md` | Plano e regras do Firebase (login, Firestore, App Check, cobrança) |
| `corridaplus-security-review.md` | Revisão de segurança (OWASP Top 10) |
| `LICENSE.md` | Licença de uso (uso comercial reservado ao autor) |

A app usa bibliotecas externas por CDN: SDK do Firebase, SheetJS (exportação Excel) e Google Fonts.

---

## 🚀 Como correr

1. **Publicar o ficheiro** num repositório, por exemplo no GitHub Pages (a app é um único `corridaplus.html`).
2. **Firebase** (opcional, para login e sincronização):
   - Criar um projeto no [Firebase Console](https://console.firebase.google.com).
   - Ativar **Authentication → Google** e criar a base de dados **Firestore**.
   - Colocar a configuração da app web (`firebaseConfig`) no ficheiro.
   - Publicar as regras do Firestore (ver `corridaplus-firebase-plan.md`). Sem elas, os dados não são lidos nem gravados.
3. **Planilha / Drive** (opcional, para cópias diárias):
   - Criar um script Apps Script com as funções de leitura e escrita de um ficheiro JSON no Google Drive.
   - Publicar como aplicação web e copiar o URL (termina em `/exec`).
   - Colar o URL em **Modo Dev → Sincronização**. Fica guardado na conta Google (ver a versão 0.4.33 na documentação).
   - ⚠️ O URL dá acesso aos dados. Não o partilhes.

Sem Firebase, a app funciona em **modo básico**, guardando os dados só no aparelho.

---

## 📱 Instalar no telemóvel

Abre o site no navegador (Safari ou Chrome), usa **Partilhar** e escolhe **"Adicionar ao ecrã principal"** (iPhone) ou **"Instalar app"** (Android). O ícone fica junto das outras apps, sem barra de endereço.

---

## ⚠️ Estado

- Versão de **teste**: os utilizadores entram com conta Google.
- A **cobrança por assinatura** (Stripe) ainda não está ligada. Ver a Fase 3 em `corridaplus-firebase-plan.md`.
- A app é mantida pelo autor. Uso pessoal e não comercial é permitido; o uso comercial é exclusivo do autor (ver `LICENSE.md`).
