# Automação de Testes de API: ServeRest

[![Testes de API](https://github.com/Felipee1236/Testes-API-AI-R/actions/workflows/api.yml/badge.svg)](https://github.com/Felipee1236/Testes-API-AI-R/actions/workflows/api.yml)

Testes automatizados do CRUD de usuários da API [ServeRest](https://serverest.dev), com autenticação JWT, desenvolvidos como desafio técnico. Os testes rodam automaticamente no GitHub Actions, em um ServeRest local e na API pública, e os relatórios são publicados como artefato.

> A coleção cria e remove os próprios dados, então pode ser executada quantas vezes for necessário.

## Stack

| Ferramenta | Uso |
| --- | --- |
| [Postman](https://www.postman.com) | Escrita e organização dos testes |
| [Newman](https://github.com/postmanlabs/newman) | Execução da coleção pela linha de comando |
| newman-reporter-htmlextra | Relatório HTML interativo |
| JUnit (reporter nativo do Newman) | Relatório padrão para ferramentas de CI |
| GitHub Actions | Integração contínua |
| [ServeRest em Docker](https://hub.docker.com/r/paulogoncalvesbh/serverest) | API local e isolada para a pipeline |

## Pré-requisitos

- [Node.js](https://nodejs.org) 22 ou superior (a pipeline usa a versão do `.nvmrc`, 24 LTS)
- Docker (opcional), para subir o ServeRest local em container
- Postman (opcional), para visualizar e editar a coleção

## Instalação e execução

```bash
git clone https://github.com/Felipee1236/Testes-API-AI-R.git
cd Testes-API-AI-R
npm ci
```

**Contra a API pública** ([serverest.dev](https://serverest.dev)):

```bash
npm test
```

**Contra o ServeRest local**, com a API rodando em outro terminal:

```bash
npm run serverest          # ou: docker run -p 3000:3000 paulogoncalvesbh/serverest:3.2.2
npm run test:local
```

Ao final, os relatórios ficam em `reports/`:

- `relatorio.html`: relatório interativo, com cada requisição, resposta e verificação
- `junit.xml`: formato padrão lido por ferramentas de CI

**Pelo Postman:** **Import** → selecione os arquivos de `collections/` e `environments/` → escolha o ambiente **ServeRest (serverest.dev)** ou **ServeRest (local)** → **Run collection**, com *Delay* de 600 ms (veja [limite de requisições](#limite-de-100-requisições-por-minuto)). A coleção deve ser executada inteira e na ordem, porque cada pasta prepara os dados da seguinte.

## Estrutura do projeto

```
collections/
└── serverest-usuarios.postman_collection.json   # requisições e testes
environments/
├── serverest.postman_environment.json           # URL da API pública
└── local.postman_environment.json               # URL do ServeRest local
.github/
└── workflows/api.yml                            # pipeline de CI
reports/                                         # relatórios gerados (fora do Git)
```

## Casos de teste

As pastas da coleção seguem o ciclo de vida do usuário: cadastro, consulta, alteração, autenticação e limpeza. Além das verificações de cada caso, **todas** as requisições verificam que o tempo de resposta fica abaixo de 3 segundos e que a resposta é JSON.

### `POST /usuarios`: cadastro

| ID | Cenário | Resultado esperado |
| --- | --- | --- |
| CT01 | Cadastrar administrador (`administrador: "true"`) com dados válidos | 201, mensagem de sucesso e `_id` com 16 caracteres alfanuméricos |
| CT02 | Cadastrar usuário comum (`administrador: "false"`) com dados válidos | 201, mensagem de sucesso e `_id` |
| CT03 | E-mail já cadastrado | 400, "Este email já está sendo usado" |
| CT04 | Corpo vazio | 400, mensagem de obrigatoriedade para cada um dos 4 campos |
| CT05 | Campos presentes, mas em branco | 400, "não pode ficar em branco" para cada campo |
| CT06 | Tipos inválidos: número nos textos e `administrador: true` (booleano) | 400, "deve ser uma string" e administrador recusado |
| CT07 | E-mail em formato inválido | 400, "email deve ser um email válido" |
| CT08 | `administrador` diferente de `"true"`/`"false"` | 400, "administrador deve ser 'true' ou 'false'" |
| CT09 | Envio de `_id` no corpo (mass assignment) | 400, "_id não é permitido" |
| CT10 | JSON malformado | 400 com mensagem da API, sem erro interno |

### `GET /usuarios`: listagem

| ID | Cenário | Resultado esperado |
| --- | --- | --- |
| CT11 | Listar todos | 200, contrato validado com JSON Schema, `quantidade` igual ao total de itens e usuários da execução presentes |
| CT12 | Filtrar por e-mail | Exatamente o usuário cadastrado |
| CT13 | Filtrar por parte do nome, em minúsculas | Exatamente o usuário cadastrado (busca parcial e sem diferenciar maiúsculas) |
| CT14 | Filtrar por `administrador=false` | Apenas usuários comuns: inclui o comum e não o administrador da execução |
| CT15 | Combinar os 5 filtros (`_id`, `nome`, `email`, `password`, `administrador`) | Exatamente o usuário cadastrado |
| CT16 | Filtro sem correspondência | 200, `quantidade` 0 e lista vazia |
| CT17 | Filtros com valores inválidos (e-mail e administrador) | 400, mensagem para cada parâmetro |
| CT18 | Parâmetro de filtro não previsto | 400, "cpf não é permitido" |

### `GET /usuarios/{id}`: busca por id

| ID | Cenário | Resultado esperado |
| --- | --- | --- |
| CT19 | Usuário existente | 200, contrato validado e todos os campos iguais aos cadastrados |
| CT20 | Id inexistente, em formato válido | 400, "Usuário não encontrado" |
| CT21 | Id fora do formato de 16 caracteres alfanuméricos | 400, mensagem com o formato esperado |

### `PUT /usuarios/{id}`: alteração

| ID | Cenário | Resultado esperado |
| --- | --- | --- |
| CT22 | Alterar nome e senha | 200 e nova consulta confirmando todos os campos gravados |
| CT23 | Trocar o e-mail pelo de outro usuário | 400, "Este email já está sendo usado" |
| CT24 | Corpo vazio | 400, mensagem para cada campo obrigatório |
| CT25 | E-mail e administrador inválidos | 400, mensagem para cada campo |
| CT26 | Id inexistente | 201: a API cadastra um novo usuário, confirmado por consulta |
| CT27 | Id inexistente com e-mail já cadastrado | 400, "Este email já está sendo usado" |

### Autenticação JWT: login e rota protegida

| ID | Cenário | Resultado esperado |
| --- | --- | --- |
| CT28 | Login do administrador | 200 e JWT válido: formato `Bearer`, algoritmo HS256, e-mail no payload e validade de 600 segundos |
| CT29 | Senha incorreta | 401, "Email e/ou senha inválidos", sem token |
| CT30 | E-mail não cadastrado | 401 com a **mesma** mensagem da senha incorreta, sem revelar quais e-mails existem |
| CT31 | Login com corpo vazio | 400, mensagem para e-mail e senha |
| CT32 | Login do usuário comum | 200 e token no formato `Bearer` |
| CT33 | Rota protegida sem token | 401 |
| CT34 | Rota protegida com token adulterado (payload alterado sem nova assinatura) | 401 |
| CT35 | Usuário comum em rota exclusiva de administrador | 403, "Rota exclusiva para administradores" |
| CT36 | Administrador em rota protegida | 201, produto cadastrado |

A pasta **Limpeza** remove, ao final, o produto, o usuário comum, o usuário criado pelo PUT e o administrador. Ela também tem verificações, para que um resíduo na API nunca passe despercebido.

## Decisões técnicas

### Endpoints `/usuarios` em vez de `/users`

O enunciado descreve os endpoints como `/users` e sugere a API ServeRest, que expõe o mesmo CRUD em português (`/usuarios`), com os mesmos campos obrigatórios: `nome`, `email`, `password` e `administrador` (string). Os testes seguem a API real.

### Autenticação JWT

No ServeRest, as rotas de usuários são públicas; o token JWT é exigido nas rotas de administração, como o cadastro de produtos. Para cobrir o requisito de autenticação, a pasta **Autenticação JWT**:

1. faz login com o administrador e com o usuário comum criados na execução
2. valida o token: prefixo `Bearer`, três partes, algoritmo HS256, e-mail do usuário no payload e validade de 600 segundos, como documentado pela API
3. comprova que a rota protegida recusa requisição sem token e com token adulterado (401), recusa usuário sem perfil de administrador (403) e aceita o administrador (201)

O token adulterado mantém cabeçalho e assinatura originais e só prorroga a expiração no payload. Se a API não validasse a assinatura, esse token seria aceito, então o teste prova que a validação existe.

### Limite de 100 requisições por minuto

O Newman espera **600 ms entre requisições** (`--delay-request 600`): como 60.000 ms ÷ 600 ms = 100, o ritmo nunca passa de 100 requisições por minuto, mesmo que a coleção cresça.

Se a API responder **HTTP 429** (limite excedido), uma verificação da coleção falha com uma mensagem explicando o motivo, em vez de deixar só uma sequência de falhas sem causa aparente.

O limite não é testado ativamente, porque isso exige disparar mais de 100 requisições em um minuto. Na API pública, seria um teste de carga em ambiente compartilhado, que o ServeRest proíbe e bloqueia. A instância pública também tem um limite próprio: no código do ServeRest 3.2.2, são 300 requisições a cada 30 segundos por IP. Esse teste fica para um ambiente dedicado.

### Dois ambientes: local e público

Os testes rodam em dois ambientes, definidos em `environments/`, e só a variável `baseUrl` muda:

- **Local** (`http://localhost:3000`): ServeRest 3.2.2 rodando na própria máquina ou em container, com dados isolados e sem depender de rede externa. Uma falha aqui indica problema nos testes ou na API, nunca instabilidade de terceiros.
- **Público** (`https://serverest.dev`): a API sugerida no desafio, compartilhada com outros usuários.

### Massa de dados independente

Não existe usuário fixo. Cada execução cadastra os próprios usuários, com e-mail único no domínio reservado `example.com` ([RFC 2606](https://www.rfc-editor.org/rfc/rfc2606)), e a pasta **Limpeza** remove tudo o que foi criado. O Newman continua a execução mesmo quando um teste falha, então a limpeza sempre roda. Dados usados em uma única requisição ficam em variáveis locais; só o que é compartilhado entre requisições (ids, credenciais e tokens) fica em variáveis da coleção.

### Comportamento de "upsert" no PUT

Pela documentação da API, um `PUT` com id inexistente cadastra um novo usuário em vez de retornar erro. Esse comportamento é testado explicitamente (CT26 e CT27), e o usuário criado é removido na limpeza.

### Contrato com JSON Schema

O schema do usuário fica em uma variável da coleção (`schemaUsuario`) e é reutilizado na listagem e na busca por id. Ele não aceita campos além dos documentados (`additionalProperties: false`), para que um campo novo na resposta, como um dado sensível exposto por engano, seja detectado.

### Dados sensíveis

- A única configuração é a URL base, que não é sensível. Credenciais de teste são geradas em tempo de execução e não ficam em nenhum arquivo versionado.
- O relatório HTML oculta o header `Authorization` e a resposta dos logins, para que nenhum token seja publicado no artefato da pipeline.
- Os tokens deixam de valer ao fim da execução, porque os usuários são excluídos.

## Integração contínua

A pipeline fica em `.github/workflows/api.yml` e roda a cada `push`, em pull requests para a `main` e manualmente, pela aba **Actions**. Uma matriz executa os testes nos dois ambientes, em paralelo e de forma independente:

| Job | Ambiente |
| --- | --- |
| Testes de API (local) | ServeRest 3.2.2 em container Docker, iniciado pela própria pipeline |
| Testes de API (serverest) | API pública, `https://serverest.dev` |

Etapas de cada job:

1. Instala o Node.js da versão do `.nvmrc` e as dependências com `npm ci`
2. No ambiente local, sobe o ServeRest em container e aguarda a API responder
3. Executa a coleção com o Newman: qualquer verificação que falhe reprova o job
4. Publica os relatórios HTML e JUnit como artefato, **mesmo quando algum teste falha**

**Onde ver o relatório:** aba **Actions** → execução desejada → seção **Artifacts** → `relatorios-api-local` ou `relatorios-api-serverest`. Basta extrair e abrir o `relatorio.html` no navegador.

Boas práticas aplicadas:

- permissão mínima para o token da pipeline (`contents: read`)
- limite de 10 minutos por job
- cancelamento automático de execuções antigas da mesma branch

## Autor

Felipe Campos de Souza · [LinkedIn](https://linkedin.com/in/felipe-campos-de-souza) · [GitHub](https://github.com/Felipee1236)
