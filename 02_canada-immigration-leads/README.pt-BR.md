# 🍁 Automação completa de captura de leads e envio em massa para empresa de imigração canadense

Criei uma automação completa de leads para 3 eventos no próximo mês que é responsável por tratar todo o processo de registro e acompanhamento por email e WhatsApp que acontece em um formulário de anúncio do Meta e no site do cliente. Aqui está como fiz e quais tecnologias usei:

## ⚙️ 1. Configurando os ambientes de captura de leads

### 🖥️ 1.1 Adicionando um formulário ao site para captura de leads

Eles usam WordPress, então simplesmente adicionei um widget HTML através da interface do Elementor contendo o que eu chamaria basicamente de uma página web completa (HTML, CSS e JS) que tem como propósito principal mostrar um formulário com campos de nome, email, telefone e local do evento e disparar uma requisição para uma automação n8n (será mostrada depois). Fiz isso para evitar baixar desnecessariamente plugins de popup, já que o site deles é muito básico em termos de funcionalidade. Na página principal há 3 seções, cada uma responsável por cada evento que vai acontecer em cidades diferentes. Sempre que você clica em uma delas, este popup será exibido já configurado para enviar um payload com a cidade escolhida em mente. O usuário também pode escolher qual evento gostaria de participar através de um select, independentemente de qual seção clicou.

- **Este é o popup:**

![Alt text](https://raw.githubusercontent.com/renato-fb/client-automation-use-cases/refs/heads/main/02_canada-immigration-leads/assets/screenshots/expo-form.jpg)

### ⓕ 1.2 Configurando uma conta dev do Meta para enviar uma requisição contendo informações de lead capturadas em um formulário de anúncio para um webhook no n8n.

O título é bem autoexplicativo. Isso também foi bem direto, embora eu tenha tido um pouco de dificuldade para entender como a API do Meta funciona. Descobri que eles têm uma documentação muito confusa sobre captura de leads, pelo menos no aspecto de formulários. Isso pode ser também porque esta é a primeira vez que fiz uma integração com o Meta.

- **Esta é a automação responsável por fazer isso:**
  ![Alt text](https://raw.githubusercontent.com/renato-fb/client-automation-use-cases/refs/heads/main/02_canada-immigration-leads/assets/screenshots/captacao-fb-leads-expo.jpg)

## ⭐ 2. Criando as automações reais para processamento de leads

Nesta parte, usei n8n (responsável pela maior parte da automação), Chatwoot (para acompanhar os números para os quais a automação enviou mensagens) em combinação com Evolution API e um número de telefone para enviar mensagens, e Supabase com PostgreSQL DB. Tudo isso é auto-hospedado em um VPS que eu também configurei.

### 2.1 O workflow principal para processamento de leads

- **Como funciona (ciclo de vida):**
  -> Usuário ativa webhook com dados do formulário \
  -> Dados são normalizados em um formato global para o resto da automação, independentemente de sua fonte e formato. \
  -> Supabase verifica se o usuário já existe no banco de dados através de uma chave única (nome, email, telefone, id_evento <- Se todos estes coincidirem, o workflow é interrompido) \
  -> Um código QR é gerado e salvo no banco de dados em formato base64 através de uma API pública (https://api.qrserver.com/v1/) com o parâmetro de consulta do ID do usuário. Este código QR deve ser escaneado por um membro da equipe na entrada nos dias do evento apenas para determinar quem compareceu, já que a entrada é gratuita. (A plataforma que será usada para escaneamento do código QR também foi criada por mim, como será mostrado depois) \
  -> Uma mensagem de texto e uma de 3 imagens são enviadas (cada uma contendo textos de cidade diferentes de acordo com o evento escolhido, já que todos os 3 são em cidades diferentes). A mensagem de texto e imagem são enviadas ao lead da seguinte forma:

![Alt text](https://raw.githubusercontent.com/renato-fb/client-automation-use-cases/refs/heads/main/02_canada-immigration-leads/assets/screenshots/main-automation.jpg)

Todas as requisições são enviadas usando Evolution API \
-> A cidade escolhida é filtrada para buscar a URL da imagem correta e a descrição do email para ser anexada ao email que também será enviado ao usuário
\
-> Finalmente, o status das mensagens enviadas é salvo no banco de dados em formato JSON

📄 _Consulte a seção Arquitetura do DB para referência se necessário [aqui](#arquitetura-do-db)_

**_Todos os diferentes nós que você pode ver na imagem que não mencionei ainda estão em desenvolvimento e serão responsáveis por funcionalidades futuras que ainda estão no backlog_**

### 2.2 O workflow responsável por capturar leads de formulários do Meta

![Alt text](https://raw.githubusercontent.com/renato-fb/client-automation-use-cases/refs/heads/main/02_canada-immigration-leads/assets/screenshots/captacao-fb-leads-expo.jpg)

Este é um workflow relativamente simples. Ele recebe a requisição no webhook do Meta sempre que um formulário de anúncio é preenchido, e os dados pertinentes a esse lead são enviados para o workflow principal para serem totalmente registrados e processados.

## 📲 3. Página para escaneamento de código QR

Quando chega o dia do evento, será necessário que os participantes tenham seu código QR (que foi gerado anteriormente e enviado para seu WhatsApp) escaneado. Esta página foi criada para a pessoa que gerencia a entrada do evento poder escanear esses códigos QR. Aqui está a interface:

![Alt text](https://raw.githubusercontent.com/renato-fb/client-automation-use-cases/refs/heads/main/02_canada-immigration-leads/assets/screenshots/qr-admin.png)

Assim que o código QR é lido, uma requisição é feita para uma automação que define o status do lead no banco de dados como "checked-in". O banco de dados também tem uma coluna que conta quantas vezes o código QR foi lido para poder rastrear quantas pessoas entraram, mesmo que o mesmo código QR seja escaneado várias vezes (ex: quando é compartilhado com amigos ou familiares que estarão participando do mesmo evento). Não vou mostrar esta automação já que é bem simples.

## Arquitetura do DB

- **`id`**: Identificador único (UUID) para cada registro de inscrição.
- **`nome`**: Nome completo do participante.
- **`telefone`**: Número de telefone do participante (usado para despachos do WhatsApp).
- **`email`**: Endereço de email do participante (usado para despachos de email).
- **`created_at`**: Timestamp de quando o lead foi registrado no banco de dados.
- **`id_evento`**: Identificador que vincula o participante a um evento específico.

---

### Colunas de Controle de Despacho (`disp_*`)

Essas colunas armazenam um objeto JSON (ou `null`) que registra o status dos despachos de comunicação (ex: `{ "status_wpp": true, "status_email": true, ... }`).

- **`disp_inscricao`**: Status do despacho enviado no momento da inscrição.
- **`disp_dia_pos_inscricao`**: Status do despacho enviado 1 dia após a inscrição.
- **`disp_30_dias_antes`**: Status do despacho enviado 30 dias antes do evento.
- **`disp_1_semana_antes`**: Status do despacho enviado 1 semana antes do evento.
- **`disp_3_dias_antes`**: Status do despacho enviado 3 dias antes do evento.
- **`disp_1_dia_antes`**: Status do despacho enviado 1 dia antes do evento.
- **`disp_dia_evento`**: Status do despacho enviado no dia do evento.

---

### Colunas de Check-in e Origem

- **`qr_base64`**: Uma string base64 representando o código QR único do participante para check-in.
- **`status`**: O status geral do participante no evento (ex: 'confirmed', 'pending', 'checked-in').
- **`qtd_scan`**: Um contador numérico de quantas vezes o `qr_base64` foi escaneado.
- **`origem`**: A fonte de aquisição do lead (ex: 'meta', 'wordpress', 'ci').

## 🛠️ Tecnologias Utilizadas

![n8n](https://img.shields.io/badge/n8n-FF6D5A?style=for-the-badge&logo=n8n&logoColor=white) ![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=for-the-badge&logo=supabase&logoColor=white) ![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white) ![Evolution API](https://img.shields.io/badge/Evolution_API-25D366?style=for-the-badge&logo=whatsapp&logoColor=white) ![Chatwoot](https://img.shields.io/badge/Chatwoot-FF6B6B?style=for-the-badge&logo=chatwoot&logoColor=white) ![WordPress](https://img.shields.io/badge/WordPress-21759B?style=for-the-badge&logo=wordpress&logoColor=white) ![Meta API](https://img.shields.io/badge/Meta_API-1877F2?style=for-the-badge&logo=meta&logoColor=white) ![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white) ![CSS3](https://img.shields.io/badge/CSS3-1572B6?style=for-the-badge&logo=css3&logoColor=white) ![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
