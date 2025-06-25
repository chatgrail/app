### 1. Arquitetura Conceitual do ChatGrail

*   **Módulo de Criação (Flow Studio):** Uma interface visual (React) onde os usuários podem desenhar fluxos de conversação usando nós (nodes) e arestas (edges). Esses fluxos são salvos como uma estrutura de dados (ex: JSON) no backend. Esta é a funcionalidade inspirada na **evoai.co**.
*   **Módulo de Execução (Conversation Engine):** Um serviço no backend (FastAPI) que interpreta o JSON do fluxo e simula uma conversa. Ele gerencia o estado do usuário e processa a lógica de cada nó.
*   **Módulo de Teste (Assurance Suite):** Uma interface (React) para criar "Casos de Teste". Cada caso de teste é uma sequência de inputs do usuário e as respostas esperadas do bot. Esta é a funcionalidade inspirada na **Cyara**.
*   **Módulo de Automação de Testes (Test Runner):** Um serviço no backend (FastAPI), provavelmente usando tarefas assíncronas (Celery), que executa os Casos de Teste contra o Conversation Engine, compara os resultados com os esperados e gera relatórios detalhados de sucesso, falha e performance.

---

### 2. Estrutura de Pastas e Arquivos (Monorepo)

A estrutura de monorepo é ideal aqui, mantendo o frontend e o backend no mesmo repositório para facilitar o desenvolvimento e a integração.

```
chatgrail/
├── backend/                  # Projeto FastAPI
│   ├── app/
│   │   ├── api/              # Endpoints da API (Routers)
│   │   │   ├── v1/
│   │   │   │   ├── endpoints/
│   │   │   │   │   ├── auth.py
│   │   │   │   │   ├── bots.py         # CRUD para bots
│   │   │   │   │   ├── flows.py        # CRUD para fluxos de um bot
│   │   │   │   │   ├── test_suites.py  # CRUD para suítes de teste
│   │   │   │   │   └── test_runs.py    # Endpoints para iniciar e ver resultados dos testes
│   │   │   │   └── api.py          # Agregador dos routers da v1
│   │   │   └── __init__.py
│   │   ├── core/             # Configurações e lógica central
│   │   │   ├── config.py       # Leitura de variáveis de ambiente
│   │   │   └── security.py     # Funções de JWT e hashing de senha
│   │   ├── db/               # Lógica de banco de dados
│   │   │   ├── base.py         # Base declarativa do ORM e sessão
│   │   │   └── migrations/     # Migrações do Alembic
│   │   ├── models/           # Modelos do ORM (SQLAlchemy/SQLModel)
│   │   │   ├── user.py
│   │   │   ├── bot.py
│   │   │   ├── flow.py         # Armazena o JSON do fluxo
│   │   │   ├── test_suite.py
│   │   │   ├── test_case.py
│   │   │   └── test_run.py       # Armazena os resultados de uma execução de teste
│   │   ├── schemas/          # Esquemas Pydantic para validação
│   │   │   ├── bot.py
│   │   │   ├── flow.py
│   │   │   ├── token.py
│   │   │   └── user.py
│   │   ├── services/         # Lógica de negócio principal
│   │   │   ├── conversation_engine.py  # **Coração da lógica "evoai.co"**
│   │   │   └── test_runner_service.py  # **Coração da lógica "Cyara"**
│   │   ├── workers/            # Tarefas assíncronas (Celery)
│   │   │   └── tasks.py        # Ex: execute_test_suite_task()
│   │   └── main.py           # Ponto de entrada da aplicação FastAPI
│   ├── .env.example
│   ├── Dockerfile
│   └── requirements.txt
│
├── frontend/                 # Projeto React (criado com Vite + TypeScript)
│   ├── public/
│   ├── src/
│   │   ├── api/              # Funções para chamar o backend (Axios)
│   │   │   ├── botService.ts
│   │   │   └── testService.ts
│   │   ├── assets/           # Imagens, SVGs, etc.
│   │   ├── components/       # Componentes de UI reutilizáveis (Button, Modal, etc.)
│   │   │   └── common/
│   │   │   └── layout/
│   │   ├── features/         # Módulos de funcionalidades complexas
│   │   │   ├── auth/           # Login, registro, etc.
│   │   │   ├── flow-studio/    # **Componentes da UI "evoai.co"**
│   │   │   │   ├── components/ # Node, Edge, Toolbar
│   │   │   │   ├── hooks/      # useFlowBuilder.ts
│   │   │   │   └── FlowStudio.tsx
│   │   │   ├── assurance-suite/ # **Componentes da UI "Cyara"**
│   │   │   │   ├── components/ # TestCaseEditor, ReportViewer
│   │   │   │   ├── hooks/      # useTestRunner.ts
│   │   │   │   └── AssuranceSuite.tsx
│   │   ├── hooks/            # Hooks globais (useAuth, useApi)
│   │   ├── lib/              # Funções utilitárias
│   │   ├── pages/            # Páginas da aplicação
│   │   │   ├── DashboardPage.tsx
│   │   │   ├── BotEditorPage.tsx # Página que une FlowStudio e AssuranceSuite
│   │   │   └── LoginPage.tsx
│   │   ├── routes/           # Configuração de rotas (React Router)
│   │   ├── store/            # Gerenciamento de estado global (Zustand/Redux)
│   │   └── App.tsx
│   ├── .env.example
│   ├── Dockerfile
│   ├── index.html
│   ├── package.json
│   └── tsconfig.json
│
└── docker-compose.yml        # Orquestra backend, frontend, db e redis
```

---

### 3. Descrição Técnica e Pacotes Envolvidos

#### **Backend (FastAPI)**

O backend é o cérebro do sistema, responsável pela lógica de negócio, persistência de dados e execução das tarefas pesadas.

*   **Descrição Técnica:**
    *   Uma API RESTful construída com **FastAPI** por sua alta performance, natureza assíncrona e documentação automática (Swagger/ReDoc).
    *   A autenticação será baseada em **JWT (JSON Web Tokens)**, gerenciada pelo `security.py`.
    *   O **Conversation Engine** (`conversation_engine.py`) será uma classe ou conjunto de funções que recebe um ID de fluxo e um input de usuário. Ele carrega o JSON do fluxo do banco de dados, interpreta o nó atual, processa sua lógica (enviar mensagem, pedir input, fazer chamada a API externa, etc.) e determina o próximo nó.
    *   O **Test Runner** (`test_runner_service.py`) será acionado via API. Para evitar bloqueios na requisição, ele despachará uma tarefa para o **Celery**. A tarefa (`tasks.py`) irá iterar sobre cada passo de um Caso de Teste, chamar o `Conversation Engine`, comparar a resposta real com a esperada (`assertion`), e salvar os resultados detalhados no banco de dados (`TestRun`).
    *   O banco de dados **PostgreSQL** será usado pela sua robustez e suporte a JSONB, ideal para armazenar as definições de fluxos e os resultados dos testes.

*   **Principais Pacotes Python:**
    *   `fastapi`: O framework web.
    *   `uvicorn`: O servidor ASGI para rodar o FastAPI.
    *   `pydantic`: Para validação de dados e configurações.
    *   `sqlmodel` ou `sqlalchemy`: O ORM para interagir com o banco de dados. `SQLModel` é excelente com FastAPI.
    *   `alembic`: Para gerenciar migrações de esquema do banco de dados.
    *   `psycopg2-binary`: Driver do PostgreSQL.
    *   `python-jose[cryptography]`: Para criação e validação de JWTs.
    *   `passlib[bcrypt]`: Para hashing de senhas.
    *   `celery`: Para execução de tarefas assíncronas em background (os testes).
    *   `redis`: Como message broker e backend de resultados para o Celery.

#### **Frontend (React)**

O frontend é a interface rica e interativa onde o usuário constrói e testa os bots.

*   **Descrição Técnica:**
    *   Uma Single-Page Application (SPA) construída com **React** e **TypeScript** para robustez e escalabilidade.
    *   O **Flow Studio** será o componente mais complexo, utilizando uma biblioteca como a **React Flow** para criar a interface de arrastar e soltar nós e conectá-los. Cada tipo de nó (ex: "Enviar Mensagem", "Aguardar Resposta", "Condição IF/ELSE") será um componente React customizado.
    *   O **Assurance Suite** permitirá ao usuário criar uma lista de interações. Ex: `Input: "Olá" -> Esperado: "Oi, como posso ajudar?"`.
    *   O estado global (usuário logado, bot ativo) será gerenciado com **Zustand** ou **Redux Toolkit**.
    *   A comunicação com o backend será feita via **Axios**, com o estado do servidor (caching, revalidação de dados) gerenciado de forma eficiente pela biblioteca **TanStack Query (React Query)**.
    *   Os relatórios de teste serão visualizados com gráficos e tabelas, usando uma biblioteca como a **Recharts** ou **Chart.js**.

*   **Principais Pacotes NPM/Yarn:**
    *   `react`, `react-dom`: A biblioteca base.
    *   `vite`: Ferramenta de build moderna e rápida.
    *   `typescript`: Para tipagem estática.
    *   `react-router-dom`: Para gerenciamento de rotas no lado do cliente.
    *   `axios`: Para fazer requisições HTTP à API do FastAPI.
    *   `@tanstack/react-query`: Para gerenciar estado do servidor, cache e revalidação.
    *   `zustand` ou `@reduxjs/toolkit`: Para gerenciamento de estado global do cliente.
    *   `reactflow`: **Essencial** para construir o editor de fluxos baseado em nós.
    *   `@mui/material` ou `antd`: Bibliotecas de componentes de UI para acelerar o desenvolvimento com um design consistente.
    *   `recharts` ou `chart.js`: Para visualização de dados nos relatórios de teste.
    *   `styled-components` ou `tailwindcss`: Para estilização customizada.
