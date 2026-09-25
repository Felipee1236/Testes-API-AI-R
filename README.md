# Automação de Testes de API: ServeRest

Testes automatizados do CRUD de usuários da API [ServeRest](https://serverest.dev), com autenticação JWT, desenvolvidos como desafio técnico.

> A coleção cria e remove os próprios dados, então pode ser executada quantas vezes for necessário.

## Stack

| Ferramenta | Uso |
| --- | --- |
| [Postman](https://www.postman.com) | Escrita e organização dos testes |
| [Newman](https://github.com/postmanlabs/newman) | Execução da coleção pela linha de comando |
| newman-reporter-htmlextra | Relatório HTML interativo |
| JUnit (reporter nativo do Newman) | Relatório padrão para ferramentas de CI |

## Pré-requisitos

- [Node.js](https://nodejs.org) 22 ou superior (a versão recomendada, 24 LTS, está no `.nvmrc`)
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
reports/                                         # relatórios gerados (fora do Git)
```

## Casos de teste

As pastas da coleção seguem o ciclo de vida do usuário: cadastro e limpeza. Além das verificações de cada caso, **todas** as requisições verificam que o tempo de resposta fica abaixo de 3 segundos e que a resposta é JSON.

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

A pasta **Limpeza** remove, ao final, o usuário comum e o administrador. Ela também tem verificações, para que um resíduo na API nunca passe despercebido.

## Decisões técnicas

### Endpoints `/usuarios` em vez de `/users`

O enunciado descreve os endpoints como `/users` e sugere a API ServeRest, que expõe o mesmo CRUD em português (`/usuarios`), com os mesmos campos obrigatórios: `nome`, `email`, `password` e `administrador` (string). Os testes seguem a API real.

### Limite de 100 requisições por minuto

O Newman espera **600 ms entre requisições** (`--delay-request 600`): como 60.000 ms ÷ 600 ms = 100, o ritmo nunca passa de 100 requisições por minuto, mesmo que a coleção cresça.

Se a API responder **HTTP 429** (limite excedido), uma verificação da coleção falha com uma mensagem explicando o motivo, em vez de deixar só uma sequência de falhas sem causa aparente.

O limite não é testado ativamente, porque isso exige disparar mais de 100 requisições em um minuto. Na API pública, seria um teste de carga em ambiente compartilhado, que o ServeRest proíbe e bloqueia. A instância pública também tem um limite próprio: no código do ServeRest 3.2.2, são 300 requisições a cada 30 segundos por IP. Esse teste fica para um ambiente dedicado.

### Dois ambientes: local e público

Os testes rodam em dois ambientes, definidos em `environments/`, e só a variável `baseUrl` muda:

- **Local** (`http://localhost:3000`): ServeRest 3.2.2 rodando na própria máquina ou em container, com dados isolados e sem depender de rede externa. Uma falha aqui indica problema nos testes ou na API, nunca instabilidade de terceiros.
- **Público** (`https://serverest.dev`): a API sugerida no desafio, compartilhada com outros usuários.

### Massa de dados independente

Não existe usuário fixo. Cada execução cadastra os próprios usuários, com e-mail único no domínio reservado `example.com` ([RFC 2606](https://www.rfc-editor.org/rfc/rfc2606)), e a pasta **Limpeza** remove tudo o que foi criado. O Newman continua a execução mesmo quando um teste falha, então a limpeza sempre roda. Dados usados em uma única requisição ficam em variáveis locais; só o que é compartilhado entre requisições (ids e credenciais) fica em variáveis da coleção.

### Dados sensíveis

A única configuração é a URL base, que não é sensível. Credenciais de teste são geradas em tempo de execução e não ficam em nenhum arquivo versionado.

## Autor

Felipe Campos de Souza · [LinkedIn](https://linkedin.com/in/felipe-campos-de-souza) · [GitHub](https://github.com/Felipee1236)
