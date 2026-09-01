# SportSphere — React + Python

📄 **Landing page:** [`docs/index.html`](docs/index.html) — página de apresentação do projeto, estática e sem dependências. Para a publicar, ativa o GitHub Pages em *Settings → Pages* com a source `main` / `/docs`.

Assistente desportivo local com RAG sobre `local-docs/`, deteção automática
de desporto, e respostas em streaming via Ollama. Arquitetura dividida em:

- **`backend/`** — FastAPI. Toda a lógica de Python (RAG, scoring, histórico,
  extração de ficheiros, gestão de documentos) portada do `streamlit_app.py`
  original e expandida.
- **`frontend/`** — React (Vite). UI com streaming em tempo real, tema
  claro/escuro, Markdown, e gestão visual de documentos.
- **`local-docs/`** — os documentos locais, versionados no repositório
  (estrutura por desporto: `soccer/`, `basketball/`, `nfl/`, `tennis/`,
  `cricket/`, `formula1/`, `olympics/`, mais qualquer pasta custom criada
  pela UI).

## Porque dois processos?

O React só corre no browser — não tem permissões para aceder a ficheiros
locais ou chamar o Ollama diretamente. Por isso toda a lógica (RAG, OCR,
chamadas ao modelo) vive no backend, que o frontend contacta via HTTP:

```
React (frontend)  ->  FastAPI (backend)  ->  Ollama (modelo local)
```

Há duas formas de correr isto: **Docker** (tudo orquestrado, recomendado)
ou **manual** (três terminais, mais controlo/debug rápido). As duas
funcionam com o mesmo código, só muda a forma de arrancar os processos.

---

## Opção A — Docker (recomendado)

Tudo corre dentro de containers: Ollama, backend e frontend. Não precisas
de instalar Python, Node nem o Ollama no teu sistema.

Pré-requisito: [Docker Desktop](https://www.docker.com/products/docker-desktop/)
instalado e a correr.

```powershell
# 1. Cria o ficheiro de histórico vazio (só na primeira vez,
#    senão o Docker monta uma pasta em vez de um ficheiro)
Set-Content backend\chat_history.json "[]"

# 2. Constrói e arranca tudo (demora uns minutos na primeira vez)
docker compose up -d --build

# 3. Puxa o modelo para dentro do container do Ollama (só na primeira vez)
docker compose exec ollama ollama pull qwen2.5:3b
```

Confirma que está no ar: `http://localhost:8000/api/health`

Abre a app em: `http://localhost:5173`

**Comandos úteis:**

```powershell
docker compose logs -f backend     # ver logs em tempo real
docker compose down                # parar tudo
docker compose up -d --build       # reconstruir depois de alterares código
```

**Persistência:** `local-docs/` e `backend/chat_history.json` são montados
como volumes — qualquer alteração feita pela app fica guardada no teu
disco, mesmo que pares ou reconstruas os containers. O modelo do Ollama
fica guardado no volume `ollama_data` (não precisas de repetir o `pull`
a menos que apagues esse volume).

---

## Opção B — Manual (sem Docker)

Útil para debug rápido (sem rebuild de imagem a cada alteração) ou se
preferires não usar Docker.

### 1. Ollama (obrigatório antes de tudo)

O Ollama tem de estar a correr **antes** de arrancar o backend, caso contrário
o modelo aparece como OFFLINE na app.

```bash
ollama serve
```

Deixa este terminal **sempre aberto**. Noutro terminal, instala o modelo se
ainda não o tiveres:

```bash
ollama pull qwen2.5:3b
```

### 2. Backend

Noutro terminal:

```bash
cd backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # ajusta OLLAMA_MODEL / LOCAL_DOCS_PATH se preciso
uvicorn main:app --reload --port 8000
```

Confirma que está no ar: `http://127.0.0.1:8000/api/health`

> Nota: a extração de texto de imagens (OCR) usa `pytesseract`, que precisa
> do binário Tesseract instalado no sistema (não só do pacote Python).
> Se correres sem Docker, instala-o separadamente — caso contrário o OCR
> cai automaticamente para o fallback de modelo de visão via Ollama.

### 3. Frontend

Noutro terminal:

```bash
cd frontend
npm install
npm run dev
```

Abre o URL que o Vite mostrar (normalmente `http://localhost:5173`; se
a porta estiver ocupada, o Vite avança para 5174/5175 automaticamente —
o backend já aceita CORS dessas três portas).

---

## Estrutura

```
SportSphere/
├── .gitignore
├── docker-compose.yml      # orquestra ollama + backend + frontend
├── docs/
│   └── index.html          # landing page do projeto (GitHub Pages)
├── backend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── main.py            # endpoints FastAPI (chat SSE, historico, upload,
│   │                         health, gestao de documentos)
│   ├── rag.py              # RAG, scoring, deteção de desporto, system
│   │                         prompt, streaming Ollama, cache de indice (60s)
│   ├── docs_manager.py     # listar/criar/apagar pastas e ficheiros de
│   │                         local-docs/ a partir da UI
│   ├── file_extraction.py  # PDF / DOCX / OCR de imagens (uploads do utilizador)
│   ├── storage.py          # historico de conversas em chat_history.json
│   └── requirements.txt
├── frontend/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── index.html
│   └── src/
│       ├── App.jsx                 # estado global, troca entre as 3 views
│       ├── api.js                  # fetch + parsing do streaming SSE
│       ├── index.css               # tokens de tema (claro/escuro), base
│       ├── assets/
│       │   ├── logo.png            # logo para fundos claros
│       │   └── logo-dark.png       # logo para a sidebar (sempre escura)
│       └── components/
│           ├── Sidebar.jsx/css      # nova conversa, historico, navegacao,
│           │                          toggle de tema, estado do modelo
│           ├── Welcome.jsx/css      # ecra inicial com sugestoes
│           ├── ChatPanel.jsx/css    # mensagens (Markdown), copiar,
│           │                          editar/reenviar, parar, ver fontes
│           ├── InputBar.jsx/css     # input + anexar + enviar/parar
│           ├── Architecture.jsx/css # pagina "Arquitetura do Sistema"
│           └── DocsLibrary.jsx/css  # pagina "Documentos Locais"
└── local-docs/              # documentos por desporto (versionados no Git)
```

## Funcionalidades principais

### Chat

- Streaming token a token via **Server-Sent Events**
- Respostas em **Markdown** (listas, negrito, código, tabelas)
- **Copiar** resposta com um clique
- **Editar e reenviar** mensagens próprias (corta a conversa a partir daí)
- **Parar geração** a meio — o texto já gerado fica guardado no histórico
- **"a consultar: ..."** expansível, mostra os excertos exatos de cada
  ficheiro usados para construir a resposta
- Recusa de forma consistente quando não há contexto suficiente (frase fixa
  controlada pelo backend, não depende do modelo cumprir a instrução)

### Documentos (`/api/docs`)

- Ecrã próprio para gerir `local-docs/` sem mexer manualmente nas pastas
- Adicionar/apagar ficheiros por desporto
- Criar pastas novas (desportos fora da lista original)
- Apagar pastas custom (as 7 pastas originais não podem ser apagadas —
  estão ligadas à deteção automática de desporto em `rag.py`)
- Cache de indexação do RAG invalidada automaticamente a cada alteração

### Arquitetura

- Página dentro da própria app a explicar o sistema (RAG, OCR, pipeline,
  tech stack), recriada a partir da versão antiga em Streamlit

### Tema

- Claro/escuro com toggle na sidebar, preferência guardada no browser
- Sidebar permanece sempre escura (âncora visual fixa); o resto da app
  muda consoante o tema escolhido

## Notas técnicas

- CORS no backend aceita `localhost`/`127.0.0.1` nas portas 5173–5175
  (válido tanto em modo Docker como manual, já que o frontend Docker está
  mapeado para a porta 5173 do host)
- O histórico de chat continua em `backend/chat_history.json` (fora do Git;
  em Docker é montado como volume, por isso persiste entre `docker compose down/up`)
- A "scope" (pasta de `local-docs` ativa) é auto-detetada pelo desporto
  mencionado na pergunta, ou via `[folders: soccer]` na mensagem
- Limite de 5MB por ficheiro e tipos suportados: PDF, DOCX, TXT, MD, CSV,
  JSON, HTML, XML, YAML, RTF, e imagens (com OCR via Tesseract → Ollama
  Vision em cascata)
- Em modo Docker, as variáveis de ambiente (`OLLAMA_BASE_URL`, `OLLAMA_MODEL`,
  etc.) são definidas no `docker-compose.yml`, não no `.env` — o `.env`
  continua a ser usado apenas no modo manual