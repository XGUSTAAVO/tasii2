# API de Gerenciamento de Pedidos

Projeto prático da disciplina de **Tópicos Avançados em Sistemas de Informação II**.
O objetivo é criar uma API REST em Node.js com Express para controlar os pedidos de uma loja, separando o código em camadas e aplicando validações e testes automatizados.

## Como instalar e rodar

1. Clone o repositório e instale as dependências:
```bash
npm install
```

2. Para iniciar o servidor:
```bash
npm start
```
A API vai rodar em `http://localhost:3000`.

3. Para rodar os testes automatizados (Jest):
```bash
npm test
```

## Arquitetura e organização do projeto

O projeto foi dividido no padrão de camadas pedido na atividade (`routes`, `controllers` e `services`), utilizando armazenamento em memória (array):

- `src/routes/pedidosRoutes.js`: define os endpoints da API e chama o controller.
- `src/controllers/pedidosController.js`: recebe a requisição HTTP (`req`, `res`), pega os parâmetros/body e devolve o status code correto. Se der erro, passa para o `next(err)`.
- `src/services/pedidosService.js`: concentra toda a regra de negócio, as validações dos campos, o controle das transições de status e a lista de pedidos em memória.
- `src/app.js` e `src/server.js`: o `app.js` configura o Express, as rotas e o middleware de erro, ficando separado do `server.js` (que só dá o `app.listen`) para facilitar os testes com o Supertest sem precisar subir a porta 3000 toda vez.
- `tests/pedidos.test.js`: testes automatizados das rotas e regras usando Jest e Supertest.
- `evidencias/`: pasta com os prints dos testes feitos no Postman.

### Padrão implementado

Foi usada a **arquitetura em camadas** (Routes → Controllers → Services), parecida com um MVC sem a camada de View:

- **Routes** só mapeiam método + URL para uma função do controller.
- **Controllers** cuidam apenas do HTTP (ler `req`, responder com o status code) e não têm regra de negócio.
- **Services** têm as regras de negócio e os dados em memória, sem depender do Express.
- **Tratamento centralizado de erros**: o service lança erros com `status` (400/404) e o middleware do `app.js` transforma em resposta JSON. Qualquer erro sem `status` é tratado como erro inesperado e retorna **500**, sem ser confundido com erro de validação.

## Regras de validação e valores aceitos

Para cadastrar um pedido (`POST /pedidos`), todos os campos abaixo são obrigatórios:

- **cliente**: texto não vazio (o sistema usa `.trim()` para não aceitar apenas espaços em branco).
- **valor**: número maior que zero.
- **tipo**: aceita `"entrega"` ou `"retirada"`.
- **prioridade**: aceita `"baixa"`, `"normal"`, `"alta"` ou `"urgente"`.

Quando o pedido é criado, ele recebe um `id` numérico sequencial e o `status` inicial é sempre `"recebido"`.

### Regras de transição de status

Os status possíveis são: `recebido`, `em_preparacao`, `enviado`, `entregue` e `cancelado`.
Para evitar mudanças sem sentido (como cancelar um pedido que já foi entregue ou entregar sem preparar), a atualização (`PATCH /pedidos/:id/status`) segue estas regras:

- De `recebido` -> pode ir para `em_preparacao` ou `cancelado`.
- De `em_preparacao` -> pode ir para `enviado`, `entregue` (caso seja retirada no balcão) ou `cancelado`.
- De `enviado` -> pode ir para `entregue` ou `cancelado`.
- De `entregue` ou `cancelado` -> não pode mais mudar de status (estados finais).

Se tentar fazer uma transição fora dessa ordem ou passar um status que não existe, a API retorna **HTTP 400** explicando o motivo no JSON.

## Rotas da API e Exemplos

### 1. Criar pedido
`POST /pedidos`

**Body (JSON):**
```json
{
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal"
}
```

**Resposta (201 Created):**
```json
{
  "id": 1,
  "cliente": "Carlos Silva",
  "tipo": "entrega",
  "valor": 350,
  "prioridade": "normal",
  "status": "recebido"
}
```

### 2. Listar todos os pedidos
`GET /pedidos`

**Resposta (200 OK):**
```json
[
  {
    "id": 1,
    "cliente": "Carlos Silva",
    "tipo": "entrega",
    "valor": 350,
    "prioridade": "normal",
    "status": "recebido"
  }
]
```

### 3. Buscar pedido por ID
`GET /pedidos/:id`

- Se encontrar o ID: retorna **200 OK** com o objeto do pedido.
- Se não existir: retorna **404 Not Found** com `{"erro": "Pedido nao encontrado."}`.

### 4. Atualizar status do pedido
`PATCH /pedidos/:id/status`

**Body (JSON):**
```json
{
  "status": "cancelado"
}
```

- Se a transição for válida: retorna **200 OK** com o pedido atualizado.
- Se o status ou a transição forem inválidos: retorna **400 Bad Request**.
- Se o ID não existir: retorna **404 Not Found**.

## Testes

### Testes automatizados (Jest + Supertest)

Executar com `npm test`. O arquivo `tests/pedidos.test.js` cobre 9 cenários:

1. `POST /pedidos` válido retorna 201 com status `recebido`.
2. `POST /pedidos` com cliente vazio ou só com espaços retorna 400.
3. `POST /pedidos` com valor menor ou igual a zero retorna 400.
4. `POST /pedidos` com tipo ou prioridade inválidos retorna 400.
5. `GET /pedidos` lista os pedidos com 200.
6. `GET /pedidos/:id` retorna 200 se existir e 404 se não existir.
7. `PATCH /pedidos/:id/status` com transições permitidas retorna 200.
8. `PATCH /pedidos/:id/status` com transição inválida retorna 400 e com ID inexistente retorna 404.
9. Erro inesperado no servidor retorna 500.

### Testes manuais (Postman)

O arquivo `postman_collection.json` pode ser importado no Postman. Os prints estão na pasta `evidencias/`:

| Arquivo | Cenário |
|---|---|
| `01-criacao.jpeg` | `POST /pedidos` válido → **201 Created** |
| `02-validacao.jpeg` | `POST /pedidos` com cliente vazio e valor negativo → **400 Bad Request** |
| `03-listagem.jpeg` | `GET /pedidos` → **200 OK** |
| `04-consulta-inexistente.jpeg` | `GET /pedidos/999` → **404 Not Found** |
| `05-atualizacao.jpeg` | `PATCH /pedidos/1/status` com `cancelado` → **200 OK** |

## Uso de IA

- **Utilizou IA?** Sim, utilizei o ChatGPT/assistente de IA como apoio durante o desenvolvimento.
- **Para quais tarefas?** Usei para tirar dúvidas sobre como configurar o Jest junto com o Supertest no Express e para revisar a lógica de validação de transição de status no `pedidosService.js`.
- **Sugestão aceita, modificada ou rejeitada:**
  - **Sugestão modificada/rejeitada:** A IA inicialmente sugeriu criar pastas extras (`errors/` com uma classe `AppError` usando herança de `Error` e outra pasta `middlewares/`) além de vários arquivos separados. Eu rejeitei essa estrutura mais complexa e simplifiquei para uma função auxiliar `criarErro(mensagem, status)` dentro do próprio `pedidosService.js` e um middleware de erro direto no `app.js`. **Justificativa:** Como o projeto é pequeno e o enunciado pedia foco em `routes`, `controllers` e `services`, manter o tratamento de erro direto no `app.js` deixou o código bem mais fácil de ler, entender e explicar na avaliação, continuando a separar erros de validação (400/404) de erros inesperados do servidor (500).
