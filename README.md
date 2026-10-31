# 🚖 Corrida+

Gestão financeira para motoristas TVDE (Uber, Bolt) e corridas particulares. É uma aplicação web de ficheiro único (`corridaplus.html`), pensada para usar como app no telemóvel, com aspeto nativo e suporte ao efeito transparente ("liquid glass").

**Versão atual:** 1.0.2a (ver o histórico na Parte II de `DOCUMENTATION.md`).

---

## ✨ Funcionalidades

### Ganhos
- Registo por plataforma: **Uber**, **Bolt**, **Particular**, **Outros**, e plataformas extra criadas por ti (ex.: 99, InDrive, Free Now).
- Cada plataforma extra pode cobrar **IVA e comissão** (como Uber/Bolt) ou ser **pessoal** (sem descontos).
- Vistas **Dia, Semana, Mês** e totais por plataforma.
- **Km** (opcional): aparece nas estatísticas, com lucro por km.
- **Período** (opcional): Dia ou Noite, com gráfico "Ganhos por turno" e melhores dias.

### Despesas
- **Despesas fixas** diárias, semanais ou mensais. As mensais podem ser repartidas pelas semanas do mês, ou colocadas na primeira semana.
- **Despesas variáveis** por categoria (combustível, manutenção, lavagem, etc.), com editar e apagar.

### Impostos e lucro
- **IVA** e **comissão da plataforma**, cada um com interruptor próprio.
- Aba **Lucro** com o detalhamento completo: bruto → IVA → comissão → líquido TVDE → pessoal → combustível → despesas → **lucro líquido**.

### Estatísticas
- Vistas **Dia, Semana, Mês e Ano**, com gráficos de ganhos vs. despesas, distribuição por plataforma e por categoria.
- Análise do mês: dias trabalhados, média por dia, melhor e pior dia, média por dia da semana, ganhos acumulados e comparação com o mês anterior.
- **Metas de lucro** mensal e semanal (opcionais), com barra de progresso.
- **Resumo de ontem** (opcional): ao abrir a app pela primeira vez no dia, mostra o resultado do dia anterior.

### Conta, sincronização e cópias
- **Login com conta Google** para sincronizar os dados entre aparelhos (Firestore, em tempo real).
- **Modo básico** (sem login ou sem assinatura): até 5 ganhos, 5 despesas variáveis e 5 despesas fixas, com as configurações bloqueadas (exceto IVA e comissão).
- **Só leitura** quando a assinatura expira.
- **Planilha (Google Drive, via Apps Script)** para quem tem login: cópia diária automática ao abrir a app.
- **Exportar dados**: planilha Excel (.xlsx) para a declaração de IRS, ou arquivo de configurações (.json).
- **Apagar**: apagar só os dados, redefinir só as configurações, ou apagar tudo. Cada opção pede um código de confirmação.

### Aparência
- **Temas de cor** e **modo claro / escuro / sistema**.
- **Efeito transparente (liquid glass)** com barra de intensidade.
- **Imagem de fundo** personalizada, com véu e desfoque.
- **Ícone do app** personalizável (cor e efeito transparente), e **fundo da tela de carregamento** (igual ao tema, fundo do app ou cor do tema).
- Animações de troca de abas e seletores, com gota deslizante e rebote.

### Outros
- **Suporte** por email, com a conta e a versão já preenchidas.
- **Modo Dev** (5 toques no número da versão): importar dados, ver o log de sincronização, forçar efeitos e definir o endereço da planilha.
- **Atualizações betas** (só neste aparelho): usa `indexb.html` no mesmo servidor, para testar versões sem mexer no `index.html`.
- **Modo demonstração** (só neste aparelho): espaço limpo para testar valores, com uma conta de demonstração. Os dados ficam separados e não vão para a nuvem.

---

## 🛠️ Estrutura

| Ficheiro | Conteúdo |
|---|---|
| `corridaplus.html` | A aplicação completa (HTML, CSS e JavaScript num só ficheiro) |
| `DOCUMENTATION.md` | Documentação técnica (Parte I) e histórico de alterações por versão (Parte II) |
| `firestore.rules` | Regras do Firestore (na pasta do Firebase, não no repositório) |
| `functions/index.js` | Cloud Function que calcula o acesso (na pasta do Firebase) |
| `LICENSE.md` | Licença de uso (uso comercial reservado ao autor) |

A app usa bibliotecas externas por CDN: SDK do Firebase, SheetJS (exportação Excel) e Google Fonts.

---

## 🚀 Como correr

1. **Publicar o ficheiro** num repositório, por exemplo no GitHub Pages (a app é um único `corridaplus.html`).
2. **Firebase** (opcional, para login e sincronização): criar um projeto, ativar Authentication (Google) e Firestore, colocar a configuração da app web no ficheiro e publicar as regras e a função (ver `DOCUMENTATION.md`).
3. **Planilha / Drive** (opcional, para cópias diárias): criar um Apps Script, publicar como aplicação web e colar o URL no Modo Dev. ⚠️ O URL dá acesso aos dados. Não o partilhes.

Sem Firebase, a app funciona em **modo local**, guardando os dados só no aparelho.

---

## 📱 Instalar no telemóvel

Abre o site no navegador (Safari ou Chrome), usa **Partilhar** e escolhe **"Adicionar ao ecrã principal"** (iPhone) ou **"Instalar app"** (Android). O ícone fica junto das outras apps, sem barra de endereço.

---

## ⚠️ Estado

- Versão de **teste**: os utilizadores entram com conta Google.
- A **cobrança por assinatura** (Stripe) ainda não está ligada. Ver o roteiro em `DOCUMENTATION.md`.
- A app é mantida pelo autor. Uso pessoal e não comercial é permitido; o uso comercial é exclusivo do autor (ver `LICENSE.md`).
