# Automação de Testes de API: ServeRest

Testes automatizados do CRUD de usuários da API [ServeRest](https://serverest.dev), com autenticação JWT, desenvolvidos como desafio técnico.

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

## Instalação

```bash
git clone https://github.com/Felipee1236/Testes-API-AI-R.git
cd Testes-API-AI-R
npm ci
```

## Estrutura do projeto

```
environments/
├── serverest.postman_environment.json           # URL da API pública
└── local.postman_environment.json               # URL do ServeRest local
```

## Decisões técnicas

### Dois ambientes: local e público

Os testes rodam em dois ambientes, definidos em `environments/`, e só a variável `baseUrl` muda:

- **Local** (`http://localhost:3000`): ServeRest 3.2.2 rodando na própria máquina ou em container, com dados isolados e sem depender de rede externa. Uma falha aqui indica problema nos testes ou na API, nunca instabilidade de terceiros.
- **Público** (`https://serverest.dev`): a API sugerida no desafio, compartilhada com outros usuários.

## Autor

Felipe Campos de Souza · [LinkedIn](https://linkedin.com/in/felipe-campos-de-souza) · [GitHub](https://github.com/Felipee1236)
