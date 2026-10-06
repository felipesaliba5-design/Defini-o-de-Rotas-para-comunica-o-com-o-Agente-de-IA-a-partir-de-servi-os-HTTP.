# Especificação da API e Rotas de Comunicação — EduBot (DevAcademy)

## 1. Visão Geral do Serviço

O **EduBot** é o assistente virtual oficial de atendimento e suporte operacional dd plataforma Educacional. Projetado originalmente no ambiente **n8n**, o agente evoluiu de um protótipo acoplado à interface nativa do *Chat Trigger* para uma **arquitetura orientada a serviços**, desacoplada e acessível via requisições HTTP utilizando o nó **Webhook do n8n**.

### 1.1 Objetivo e Função do Agente
O serviço tem como missão automatizar o atendimento a alunos matriculados nos bootcamps da DevAcademy, sanando dúvidas sobre cronogramas, prazos de entrega de projetos, navegação na plataforma e suporte básico de acesso. Ao responder de forma instantânea e estritamente fundamentada em uma base de conhecimento vetorial (RAG via Supabase Vector), o EduBot reduz significativamente o volume de chamados operacionais direcionados à equipe humana de suporte.

### 1.2 Aplicações Consumidoras
A exposição da interface HTTP via Webhook permite que múltiplas aplicações *front-end* interajam de maneira padronizada com o mesmo Agente de IA:
* **Aplicação Web (Portal do Aluno):** Widget de chat integrado ao painel das aulas da DevAcademy.
* **Aplicação Mobile:** Aplicativo nativo em React Native / Flutter utilizado pelos estudantes.
* **Integração WhatsApp / Chatbots:** Middleware responsável por traduzir mensagens de texto para payloads JSON.

---

## 2. Arquitetura da Solução e Mapeamento de Operações

A transição da arquitetura substitui o acoplamento direto da interface do n8n por uma camada de serviço HTTP padronizada:

* **Fluxo de Comunicação:**  
  `Aplicação Front-end` $\rightarrow$ `HTTP POST` $\rightarrow$ `Webhook n8n` $\rightarrow$ `Edit Fields (Set)` $\rightarrow$ `AI Agent (RAG / Memory)` $\rightarrow$ `Respond to Webhook` $\rightarrow$ `JSON Response` $\rightarrow$ `Aplicação Front-end`

### 2.1 Mapeamento de Operações
| Operação | Rota Relativa | Método HTTP | Descrição e Processamento |
| :--- | :--- | :--- | :--- |
| **Enviar Mensagem / Conversar** | `/chat` | `POST` | Recebe a pergunta do aluno, recupera o contexto da sessão, realiza a busca por similaridade no Supabase Vector (se necessário) e retorna a resposta gerada pelo EduBot. |
| **Verificar Disponibilidade** | `/health` | `GET` | Endpoint de monitoramento (*healthcheck*) que confirma se o workflow do n8n e os serviços associados estão operacionais. |
| **Encerrar / Resetar Sessão** | `/reset-session` | `POST` | Permite que o *front-end* sinalize o encerramento da conversa atual, permitindo a limpeza de memória buffer no nó de histórico do n8n. |

---

## 3. Especificação Detalhada do Contrato de Comunicação (JSON)

### 3.1 Rota Principal: `POST /chat`

Esta rota é responsável pelo envio de mensagens e pela manutenção da conversação do aluno com o EduBot.

#### A. Payload de Requisição (`Request Body`)
O *front-end* deve enviar os dados formatados em JSON (`Content-Type: application/json`):

```json
{
  "session_id": "dev_student_98745_sess_01",
  "message": "Até que dia eu preciso entregar o projeto final do módulo de React?",
  "student_name": "Felipe Saliba"
}
```

##### Descrição dos Campos da Requisição
| Campo | Tipo | Obrigatoriedade | Finalidade e Descrição |
| :--- | :--- | :--- | :--- |
| `session_id` | `String` | **Obrigatório** | Identificador único da sessão de conversa. Garante a continuidade do histórico no nó `Simple Memory` do n8n. |
| `message` | `String` | **Obrigatório** | Texto enviado pelo aluno contendo a dúvida ou solicitação. Não pode ser nulo ou vazio. |
| `student_name` | `String` | Opcional | Nome do aluno. Quando fornecido, é utilizado pelo EduBot para personalizar a saudação inicial. |

#### B. Payload de Resposta com Sucesso (`Response Body` — HTTP 200 OK)
Após o processamento do `AI Agent` e a consulta à base vetorial do Supabase, o Webhook retorna o objeto JSON estruturado:

```json
{
  "status": "success",
  "session_id": "dev_student_98745_sess_01",
  "response": "Olá, Felipe! O prazo final para a entrega do projeto do módulo de React é dia 15 de novembro, até às 23h59. Lembre-se de enviar o link do repositório no painel do aluno. Se precisar de algo mais, estou por aqui!",
  "timestamp": "2026-10-06T10:15:30.000Z"
}
```

##### Descrição dos Campos da Resposta
| Campo | Tipo | Finalidade e Descrição |
| :--- | :--- | :--- |
| `status` | `String` | Indica o resultado da operação (`success` ou `error`). |
| `session_id` | `String` | Devolve o mesmo identificador recebido para confirmação do *front-end*. |
| `response` | `String` | Resposta em texto formatada pelo EduBot, respeitando as diretrizes de tom de voz, brevidade (máximo 3 parágrafos) e *guardrails*. |
| `timestamp` | `String` | Data e hora em padrão ISO 8601 da resposta gerada. |

---

## 4. Gestão de Contexto e Ciclo de Vida da Sessão (`session_id`)

Para garantir que o EduBot mantenha o contexto da conversa em interações sucessivas (ex.: o aluno perguntar sobre "aquele projeto" mencionado no turno anterior), a API adota as seguintes regras de sessão:

1. **Geração do `session_id`:** O identificador é gerado exclusivamente pela **aplicação consumidora (*front-end*)** no momento em que o usuário abre a janela de chat.
2. **Formato Padrão:** Recomenda-se a utilização do ID do usuário concatenado com timestamp ou UUID (ex.: `user123_1728210000`).
3. **Persistência no n8n:** O nó `Webhook` repassa o `session_id` ao nó `Edit Fields (Set)`, que alimenta o parâmetro `sessionId` no nó `Simple Memory` (`memoryBufferWindow`).
4. **Encerramento de Sessão:** A sessão expira automaticamente por inatividade após 30 minutos ou quando o usuário clica no botão "Nova Conversa" no *front-end*, gerando um novo `session_id`.

---

## 5. Tratamento de Erros, Exceções e HTTP Status Codes

O serviço foi projetado para responder de forma previsível e amigável em situações de falha ou requisições malformadas:

### 5.1 Matriz de Erros da API

| Erro / Situação | HTTP Status | Estrutura da Resposta JSON | Ação da Aplicação Consumidora |
| :--- | :--- | :--- | :--- |
| **Requisição Sem Mensagem** | `400 Bad Request` | `{"status": "error", "code": "MISSING_MESSAGE", "message": "O campo 'message' é obrigatório e não pode estar vazio."}` | Exibir alerta de validação no formulário do chat. |
| **Sessão Ausente** | `400 Bad Request` | `{"status": "error", "code": "MISSING_SESSION_ID", "message": "O campo 'session_id' é obrigatório para identificação do histórico."}` | Gerar um novo `session_id` e reenviar a requisição. |
| **JSON Malformado** | `400 Bad Request` | `{"status": "error", "code": "INVALID_JSON", "message": "O corpo da requisição deve ser um JSON válido."}` | Corrigir a formatação no cliente. |
| **Falha de Conexão RAG / Supabase** | `500 Internal Server Error` | `{"status": "error", "code": "SERVICE_UNAVAILABLE", "message": "Desculpe, nosso serviço de busca está temporariamente indisponível. Tente novamente em instantes."}` | Exibir mensagem de erro amigável ao aluno. |

### 5.2 Erros de Domínio vs. Erros de Protocolo HTTP
É importante destacar a diferença entre falhas técnicas e respostas de *guardrail*:
* **Dúvidas de Código ou Pedidos de Adiamento (CT02 / CT03):** Não geram erro HTTP (`400` ou `500`). A requisição retorna **HTTP 200 OK**, e o campo `response` traz a negativa educada configurada na *System Message* do EduBot.

---

## 6. Mapeamento dos Casos de Teste na Comunicação HTTP

Os cenários de teste definidos na validação comportamental do EduBot foram integrados ao contrato de comunicação HTTP:

### Caso CT01: Consulta de Prazo (Atuação do RAG)
* **Requisição:** `POST /chat` com `message: "Até que dia preciso entregar o projeto de React?"`.
* **Processamento:** O agente aciona a Tool `Supabase Vector Store`, recupera os chunks da tabela `documents` via `match_documents()`.
* **Resposta:** Retorna **HTTP 200 OK** com a data limite precisa em `response`.

### Caso CT03: Pergunta Fora de Escopo (Dúvida de Código)
* **Requisição:** `POST /chat` com `message: "Meu código tá dando o erro undefined is not a function, como arrumo?"`.
* **Processamento:** A *System Message* identifica a tentativa de obter suporte de programação e aciona o *guardrail*.
* **Resposta:** Retorna **HTTP 200 OK** informando que o EduBot não resolve dúvidas de sintaxe ou lógica.

### Caso CT04: Bug na Plataforma (Encaminhamento para TI)
* **Requisição:** `POST /chat` com `message: "A página está dando Erro 500 ao tentar entrar no curso"`.
* **Processamento:** O agente detecta uma falha de sistema exclusiva e solicita o e-mail do aluno.
* **Resposta:** Retorna **HTTP 200 OK** solicitando o e-mail para que a equipe de TI abra um chamado.

### Caso CT05: Pergunta Vaga sem Contexto do Curso
* **Requisição:** `POST /chat` com `message: "Que dia é a próxima aula ao vivo?"`.
* **Processamento:** O agente identifica a ambiguidade e interrompe a consulta ao Supabase.
* **Resposta:** Retorna **HTTP 200 OK** perguntando a qual curso ou bootcamp o aluno se refere antes de realizar a busca.
