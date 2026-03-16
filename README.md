# 🌤️ Telegram Weather Chatbot com N8N e IA (Google Gemini)

## 1. Sobre o projeto

Este projeto consiste em um assistente virtual (chatbot) integrado ao Telegram, capaz de informar a temperatura atual de qualquer cidade do Brasil. A automação foi construída utilizando o **N8N** (plataforma de automação de fluxos de trabalho) e emprega uma arquitetura robusta e escalável baseada em Docker.

O grande diferencial deste projeto é a sua **arquitetura de resiliência (fallback)**. O fluxo principal utiliza o **Agente de IA do Google Gemini** para interpretar as mensagens de forma humanizada. Caso a cota da IA seja excedida ou ocorra alguma indisponibilidade técnica, o sistema roteia o usuário automaticamente para um fluxo **determinístico de contingência**, garantindo que o usuário nunca fique sem resposta. Para contornar limitações de pesquisa por Estado no Brasil, o modo determinístico utiliza uma abordagem em duas etapas (Geocoding API seguida de Weather API).

### 1.1. Estrutura dos arquivos do projeto

* **`docker-compose.yml`**: Arquivo de orquestração de containers. Ele levanta toda a infraestrutura necessária: o editor do N8N (`n8n-editor`), um container dedicado para processar filas em background (`n8n-worker`), o banco de dados PostgreSQL (`postgres`), o Redis (`redis` - para gerenciar o Queue Mode) e o Ngrok (`ngrok` - para expor o webhook local para a internet).
* **`workflow-telegram-chatbot.json`**: O arquivo exportado contendo o fluxo visual e a lógica da automação desenvolvida no N8N.
* **`.env.example`**: Um arquivo de modelo (vitrine) que lista todas as variáveis de ambiente necessárias para a execução do projeto, sem expor chaves de API reais por questões de segurança.

---

## 2. Instalação e configuração

### 2.1. Pré-requisitos

Para executar este projeto na sua máquina, você precisará ter instalado e configurado:

* **Docker** e **Docker Compose**.
* **Telegram Bot Token**: Obtido através do `@BotFather` no Telegram.
* **Google Gemini API Key**: Obtida no Google AI Studio (opcional, mas recomendado).
* **OpenWeather API Key**: Obtida criando uma conta no site OpenWeatherMap.
* **Ngrok Auth Token**: Obtido criando uma conta no Ngrok (necessário para o webhook do Telegram se comunicar com sua máquina local).
* **NOTA**: No final deste documento há uma seção onde se pode consultar o passo-a-passo de como criar os Tokens e API keys necessários.

### 2.2. Como configurar e executar o projeto

1. Clone este repositório para a sua máquina local.
2. Na raiz do projeto, crie um arquivo chamado `.env` a partir do arquivo de exemplo:
* **Linux/Mac:** `cp .env.example .env`
* **Windows:** Copie e cole o arquivo `.env.example` e renomeie a cópia para `.env`.


4. Abra o arquivo `.env` gerado e preencha as variáveis com as suas respectivas chaves de API e tokens: substitua os valores fictícios pelas suas chaves reais. O arquivo exige as variáveis `TELEGRAM_BOT_TOKEN`, `OPENWEATHER_API_KEY`, `DEFAULT_GEMINI_API_KEY` (opcional), `WEBHOOK_URL`, `NGROK_AUTHTOKEN`, `DB_POSTGRESDB_USER` (postgres é o valor padrão) e `DB_POSTGRESDB_PASSWORD` (defina uma senha forte para o BD).
5. No terminal, dentro da pasta do projeto, execute o comando para iniciar a infraestrutura:
```bash
docker compose up -d

```
6. Quando quiser desligar a infraestrutura de servidores, no terminal, dentro da pasta do projeto, execute o comando para desligá-la:
```bash
docker compose down

```
---

## 3. Como iniciar o fluxo do N8N em Modo Produção

Após os contêineres iniciarem com sucesso, siga os passos abaixo para ativar o bot:

1. Acesse a interface do N8N no seu navegador através do endereço: `http://localhost:5678` ou via seu webhook Ngrok configurado na variável de ambiente `WEBHOOK_URL`.
2. Crie sua conta de administrador do N8N.
3. No menu lateral direito, clique em **+** e depois em **Workflow**.
4. No canto superior direito, clique no menu de três pontos (`...`) e selecione **Import from File**. Escolha o arquivo `workflow-telegram-chatbot.json`.
5. Caso as credenciais não sejam preenchidas automaticamente após a importação do arquivo, abra os nós do **Telegram**, **Google Gemini** e **OpenWeather** e adicione as suas credenciais.
    1. No menu lateral direito, clique em **+** e depois em **Credential**.
    2. Procure pela credencial do **Google Gemini (PaLM) API** e clique em **Create**.
    3. No campo **API Key**, escolha a opção **Expression**, preencha o valor com `{{ $env["DEFAULT_GEMINI_API_KEY"] }}` e salve.
    4. Adicione a credencial **Telegram API** e no campo **Access Token**, escolha a opção **Expression**, preencha o valor com `{{ $env["TELEGRAM_BOT_TOKEN"] }}` e salve.
    5. Adicione a credencial **OpenWeatherMap API** e no campo **Access Token**, escolha a opção **Expression**, preencha o valor com `{{ $env["OPENWEATHER_API_KEY"] }}` e salve.
    6. O procedimento abaixo precisará ser feito caso apareça o ícone ⚠️ nos nós do Telegram e do Gemini.
        1. Dê um duplo clique no nó que está com o ícone ⚠️.
        2. Na janela de configurações do nó, procure pelo campo "Credential to connect with". Ele provavelmente estará em branco ou com um aviso em vermelho. 
        3. Clique na caixa de seleção (dropdown) e selecione a credencial correspondente que foi criada anterioremnte.
        4. Feche a janela do nó. O ícone ⚠️ deverá desaparecer.
6. **Ative o fluxo:** No canto superior direito da tela do workflow, clique em **Publish**.
7. **Nota**: os locais dos menus, os botões e os nomes das opções podem variar de acordo com a versão do N8N que está sendo utilizada.

---

## 4. Como usar o Bot no Telegram e o agente de IA

Para utilizar o serviço, basta procurar pelo seu bot no Telegram e enviar uma mensagem. O comportamento do bot dependerá se o Agente de IA (Google Gemini) está habilitado ou se o sistema está operando no modo manual (determinístico).

### ☀️ A mágica do agente de IA (Recomendado)

A utilização do Google Gemini no projeto é **opcional**, porém traz vantagens para a experiência do usuário. O principal benefício é a **flexibilidade na entrada de dados e a compreensão de linguagem natural (NLP)**.

Com a IA ativada:

* **Tolerância a erros de digitação:** A entrada `"Joinvile, SC"` (com um 'L') é perfeitamente compreendida. A IA percebe o erro, corrige para "Joinville" e envia a requisição exata para a API de clima, evitando que o usuário receba um erro de "Cidade não encontrada".
* **Linguagem Natural:** O usuário não precisa parecer um robô. Ele pode perguntar: *"Qual é a temperatura na cidade de Curitiba, PR?"* ou *"Como tá o tempo em São Paulo, SP?"*. O agente de IA isolará apenas a cidade e o estado para realizar a pesquisa técnica por trás dos panos.

### ⚙️ Modo sem IA (Fluxo determinístico/Fallback)

Caso você opte por não configurar a chave do Gemini, ou caso a cota gratuita do Google se esgote, o sistema continuará funcionando perfeitamente através de uma lógica construída em JavaScript e expressões regulares.

No entanto, neste modo **a entrada de dados é rigorosa**. O usuário deve enviar o nome da cidade, seguido de vírgula, e a sigla do Estado, com a ortografia correta.

#### Exemplos de resultados esperados (Modo sem IA):

* **Entrada válida:** `Palmas, TO`
* **Resultado:** `🌤️ A temperatura em Palmas é de 32°C.`


* **Entrada com erro ortográfico:** `Pamas, TO`
* **Resultado:** `❌ Cidade não encontrada. Use o formato Cidade, UF (ex.: São Paulo, SP).`


* **Entrada em formato inválido (sem estado):** `Curitiba`
* **Resultado:** `❌ Cidade não encontrada. Use o formato Cidade, UF (ex.: São Paulo, SP).`

---

## Guia de criação de contas e chaves de API (Tokens)

Este guia fornece o passo a passo para criar as contas e gerar as credenciais necessárias para que o projeto funcione corretamente no seu computador local ou em infraestrutura remota.

### A. Criando o bot no Telegram e obtendo o Token
1. Abra o aplicativo do Telegram (no celular ou computador) e pesquise por **@BotFather** na barra de busca.
2. Inicie uma conversa com ele clicando em **Start** (ou enviando o comando `/start`).
3. Envie o comando `/newbot` para criar um novo robô.
4. O BotFather pedirá um **nome** para o seu bot (ex: *Bot de Clima do João*). Digite e envie.
5. Em seguida, ele pedirá um **username** (nome de usuário) único, que obrigatoriamente deve terminar com a palavra `bot` (ex: *clima_joao_bot*).
6. Após a confirmação, o BotFather enviará uma mensagem de sucesso contendo o seu **Token de Acesso HTTP API** (uma sequência longa de letras e números). 
7. Copie esse token e cole-o no seu arquivo `.env` na variável `TELEGRAM_BOT_TOKEN`.

### B. Criando conta na OpenWeather e obtendo a API Key
1. Acesse o site oficial da OpenWeather: [openweathermap.org](https://openweathermap.org/)
2. Clique em **Create an Account** no menu superior e preencha seus dados para criar uma conta gratuita.
3. Após fazer o login, clique no seu nome de usuário no canto superior direito e selecione **My API Keys**.
4. Você verá uma chave padrão (Default) já gerada. Se preferir, você pode dar um nome e gerar uma nova no campo "Create key".
5. Copie a chave (uma sequência de letras e números) e cole-a no seu arquivo `.env` na variável `OPENWEATHER_API_KEY`.
*(Atenção: A chave da OpenWeather pode levar de 1 a 2 horas para ser totalmente ativada pelos servidores deles após a criação recente de uma conta. Se o bot der erro logo nos primeiros testes, aguarde esse prazo).*

### C. Criando conta no Ngrok e obtendo o Authtoken
1. Acesse o site do Ngrok: [ngrok.com](https://ngrok.com/) e clique em **Sign up** para criar sua conta gratuita.
2. Após finalizar o cadastro e fazer o login, você será direcionado para o painel de controle (Dashboard).
3. No menu lateral esquerdo, navegue até **Tunnels** e clique em **Authtokens** (ou localize a opção "Your Authtoken" logo na página inicial).
4. Você verá o seu token de autenticação (uma string longa). Clique no botão de copiar.
5. Cole este valor no seu arquivo `.env` na variável `NGROK_AUTHTOKEN`.

### D. Criando conta no Google AI Studio e obtendo a API Key (Gemini)
1. Acesse o Google AI Studio: [aistudio.google.com](https://aistudio.google.com/) e faça login com a sua conta da Google.
2. No menu lateral esquerdo, clique no botão **Get API key** (Obter chave de API).
3. Na tela principal, clique no botão azul **Create API key**.
4. O sistema pedirá para você vincular a chave a um projeto do Google Cloud. Você pode selecionar um projeto existente ou simplesmente clicar em **Create API key in a new project** (Criar em um novo projeto).
5. Após alguns segundos, a sua chave será gerada. Copie a sequência exibida.
6. Cole o valor no seu arquivo `.env` na variável `DEFAULT_GEMINI_API_KEY`.