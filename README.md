# Triagem de Currículos com n8n e Gemini

Fluxo de automação que recebe candidaturas por Google Forms, lê o currículo em PDF e gera, com a API do Gemini, um parecer por vaga, enviado por e-mail.

A análise é apoio. **A decisão final é sempre de uma pessoa.**

> Projeto de aprendizado, construído com n8n, Google Workspace e a API do Gemini. Não é um produto pronto para uso em processos seletivos reais sem revisão jurídica e de segurança.

---

## Sumário

- [Visão geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Estrutura da planilha](#estrutura-da-planilha)
- [Pré-requisitos](#pré-requisitos)
- [Passo a passo](#passo-a-passo)
- [Prompt do Gemini](#prompt-do-gemini)
- [Problemas comuns](#problemas-comuns)
- [Limitações conhecidas](#limitações-conhecidas)
- [Dados pessoais e LGPD](#dados-pessoais-e-lgpd)
- [Próximos passos](#próximos-passos)

---

## Visão geral

1. O candidato preenche o Google Forms, escolhe a vaga em uma lista suspensa e anexa o currículo em PDF.
2. Cada envio vira uma linha na planilha, e o PDF é salvo em uma pasta do Google Drive.
3. O n8n detecta a linha nova e processa **um candidato por vez**.
4. O texto do PDF é extraído e comparado apenas com os requisitos da vaga escolhida, lidos da aba **Vaga**.
5. O Gemini devolve um parecer com resumo, pontos fortes, lacunas, adequação e nota de 0 a 10.
6. O parecer é enviado por e-mail, com o assunto `Parecer: [nome] - [vaga]`.

---

## Arquitetura

Google Forms
│
▼
Google Sheets (aba de respostas)
│ Google Sheets Trigger (rowAdded, verifica a cada minuto)
▼
Loop Over Items (1 candidato por vez)
│
▼
Download file (Google Drive, pelo ID do arquivo)
│
▼
Extract from File (PDF → texto)
│
▼
Get row(s) in sheet (aba Vaga)
│
▼
Code in JavaScript (monta a vaga e os requisitos)
│
▼
Message a model (Gemini 3.6 Flash)
│
▼
Send a message (Gmail)
│
└── volta ao Loop Over Items


**Por que o loop?** Sem ele, dois candidatos enviados juntos são processados na mesma execução, e o segundo pode receber a vaga do primeiro. O Loop Over Items garante que cada currículo passe pelo fluxo sozinho.

---

## Estrutura da planilha

A planilha tem **duas abas**.

### Aba de respostas (criada pelo Google Forms)

Recebe as candidaturas. Os nomes das colunas precisam ser exatamente estes, porque o fluxo os usa como referência:

| Coluna | Conteúdo |
|---|---|
| Carimbo de data/hora | Preenchido automaticamente |
| Nome completo | Texto |
| Telefone | Texto |
| E-mail de contato | Texto |
| Currículo em PDF | Link do arquivo no Drive |
| Vaga pretendida | Escolhida em lista suspensa |

### Aba Vaga

Guarda os requisitos. Cada **coluna** é uma vaga, e o nome do cabeçalho deve ser **idêntico** à opção do formulário.

| Campo | Cargo A | Cargo B |
|---|---|---|
| Requisitos obrigatórios | competência 1; competência 2 | competência 1; competência 2 |
| Requisitos desejáveis | diferencial 1 | diferencial 1 |
| Experiência mínima | exemplo: 1 ano em rotinas administrativas | sem experiência |
| Formação | exemplo: ensino médio completo | exemplo: graduação na área |

**Dicas para escrever os requisitos:**

- Prefira ações observáveis a adjetivos. "Atendimento ao cliente por telefone" é melhor que "proatividade", porque a IA consegue encontrar evidências no currículo.
- Evite critérios que não se relacionam ao trabalho, como idade, estado civil, origem ou aparência. O prompt já proíbe isso, mas a planilha é a primeira barreira.
- Localização só deve ser exigida se a vaga for presencial na região.

---

## Pré-requisitos

- **Conta Google** com acesso à planilha, à pasta do Drive e ao Gmail de recebimento.
- **Projeto no Google Cloud** com as APIs do **Google Sheets**, **Google Drive** e **Gmail** ativadas.
- **Tela de permissão OAuth** configurada. Enquanto o app estiver em **modo de teste**, o Google encerra as autorizações de tempos em tempos, e as credenciais precisam ser reconectadas.
- **Chave de API do Gemini**, criada no Google AI Studio.
- **n8n** rodando localmente com Docker:

```bash
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

O volume `n8n_data` guarda fluxos e credenciais. Não o apague.

---

## Passo a passo

### 1. Criar o Google Forms

- Perguntas: nome completo (resposta curta, obrigatória), telefone (resposta curta, obrigatória), e-mail (obrigatória), vaga pretendida (lista suspensa com as vagas, obrigatória) e currículo (upload de arquivo, apenas PDF, obrigatória).
- Adicione uma **descrição** que informe o uso da análise automatizada, o tratamento dos dados e o contato para exclusão. Um modelo está em `docs/texto-formulario.md`.
- Vincule o formulário à planilha: aba **Respostas** do formulário → ícone do Sheets → **Selecionar planilha existente**.

### 2. Preparar a planilha

- Confirme o nome da aba de respostas (geralmente **Respostas ao formulário 1**).
- Crie a aba **Vaga** no formato descrito acima.
- Teste enviando um candidato fictício e confira se a coluna **Vaga pretendida** aparece preenchida.

### 3. Criar as credenciais no n8n

Em **Credentials**, crie:

- **Google Sheets OAuth2 API** e **Google Sheets Trigger**
- **Google Drive OAuth2 API**
- **Gmail OAuth2 API**
- **Google Gemini(PaLM) API**, com a chave do AI Studio

Para as credenciais OAuth, use o **Client ID** e o **Client Secret** do Google Cloud, com o redirect:

http://localhost:5678/rest/oauth2-credential/callback


### 4. Importar o fluxo

1. No n8n, vá em **Workflows > Import from file** e escolha `triagem-curriculos.json`.
2. Em cada nó que usa planilha, selecione o documento e a aba.
3. Em cada nó com credencial, selecione a credencial criada no passo 3. Se aparecer vazio, a credencial ainda não foi criada.
4. Nos nós de **Google Sheets** (Trigger e Get row(s)), troque o marcador `SEU_ID_DA_PLANILHA` pelo ID real da sua planilha.
5. No nó **Send a message**, troque `seu-email@exemplo.com` pelo e-mail de recebimento.

### 5. Testar

1. Envie um candidato de teste para cada vaga, com um PDF fictício.
2. Clique em **Execute workflow** e acompanhe cada nó.
3. Confira na aba **Executions** se as execuções aparecem como **Success**.
4. Verifique se o e-mail chegou, com o assunto e a vaga corretos.
5. Teste com **dois candidatos enviados ao mesmo tempo** para confirmar que cada um recebe o parecer da própria vaga.

### 6. Publicar

Clique em **Publish**. O fluxo passa a rodar sozinho enquanto o n8n estiver aberto.

---

## Prompt do Gemini

O prompt fica no nó **Message a model**. Ele define as regras e o formato da resposta:

- avalia apenas competências, experiência e formação relacionadas ao trabalho;
- proíbe considerar idade, gênero, estado civil, origem, aparência, religião, orientação sexual, filiação política e deficiência;
- usa somente o que está no currículo, e declara quando uma informação não aparece;
- responde em português, sem Markdown, com as seções Resumo, Pontos fortes, Lacunas, Adequação, Nota e Observação.

Ao alterar o prompt, teste com pelo menos um currículo de cada tipo, incluindo um que não atenda aos requisitos. Uma IA pode ser otimista demais, e o teste precisa mostrar isso.

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| Nó com **"needs to be reconnected"** | Token expirou (app em modo de teste) | Abra a credencial e clique em **Reconnect** |
| **"Vaga não encontrada na aba Vaga"** | Nome da vaga diferente entre o formulário e o cabeçalho da planilha | Deixe os nomes idênticos, com acentos e maiúsculas |
| Campo com valor `undefined` | Nome de coluna errado na expressão | Confira o nome exato da coluna no nó **Google Sheets Trigger** |
| **"Repository not found"** no push | Nome de usuário ou do repositório diferente | Confira a URL com `git remote -v` |
| Dados antigos de teste aparecem | Dados fixados (pinned) no nó | Clique com o botão direito no nó e escolha **Unpin data** |
| Dois candidatos, só um parecer | Processamento em lote sem loop | Use o nó **Loop Over Items** com batch size 1 |
| Assunto do e-mail com o nome de teste | Expressão apontando para o gatilho, e não para o candidato do loop | Use `$('Loop Over Items')` no assunto |
| Python não funciona no nó Code | Instalação do n8n no Docker sem Python | Use JavaScript, que é nativo do nó |
| Fluxo não processa nada | Janela do terminal fechada ou fluxo não publicado | Mantenha o n8n rodando e confira o botão **Published** |

---

## Limitações conhecidas

- **Roda na máquina local.** Enquanto o computador estiver desligado ou o Docker fechado, nenhum currículo é processado.
- **Credenciais do Google expiram** enquanto o app estiver em modo de teste. Para uso contínuo, o app precisaria passar pela verificação do Google.
- **Sem validação de PDF.** Um PDF escaneado, sem texto selecionável, pode gerar um parecer vazio ou impreciso. Não há OCR neste fluxo.
- **A IA pode errar.** Pode interpretar mal uma experiência, ignorar um requisito ou ser otimista demais.
- **Sem deduplicação.** Se o mesmo candidato enviar duas vezes, serão gerados dois pareceres.
- **Sem histórico de decisão.** O fluxo registra a análise, mas não registra a decisão tomada pela pessoa responsável.

---

## Dados pessoais e LGPD

Currículos contêm dados pessoais de terceiros. Este projeto foi pensado para reduzir a exposição, mas não substitui uma avaliação jurídica.

- **Não versione dados reais.** Planilhas, PDFs, prints e pareceres não devem ser enviados ao repositório. O `.gitignore` já bloqueia `.env`, PDFs e a pasta `n8n_data`.
- **Limite o acesso.** Compartilhe a planilha e a pasta do Drive apenas com quem precisa.
- **Defina um prazo de retenção.** Apague ou anonimize os dados de candidatos que não seguem no processo.
- **Informe o candidato.** A descrição do formulário deve dizer como os dados são usados, quem os analisa e como solicitar exclusão.
- **Mantenha a decisão humana.** O parecer é apoio, e nunca deve ser usado como critério automático de eliminação.

---

## Próximos passos

- [ ] Adicionar deduplicação por e-mail ou telefone
- [ ] Registrar a decisão final na planilha (coluna **Decisão**)
- [ ] Tratar PDFs sem texto selecionável
- [ ] Hospedar o n8n em um servidor, para não depender do computador local
- [ ] Publicar o app do Google Cloud, para evitar a expiração das credenciais

---

## Aprendizado

Este projeto foi construído aos poucos, com erros que ensinaram mais do que os acertos. Os principais foram: nomes de colunas que não batem, dados de teste que ficam fixados, nós que juntam itens em vez de processá-los um a um e credenciais que expiram. As publicações sobre o processo descrevem cada um deles.