# SportSphere

Assistente desportivo com RAG local — responde só a partir dos teus documentos.

**[site do SportSphere](https://cristovao-lavarinhas.github.io/SportSphere/)**

Pergunta em português sobre regulamentos, formatos de competição ou regras de
jogo, e a resposta sai dos PDFs que tens em `local-docs/`:
**pergunta → deteta o desporto → restringe à pasta certa → responde do que lá
está.** Quando os documentos não cobrem o assunto, recusa em vez de inventar.
Nada sai da máquina — o modelo corre localmente no Ollama e não há uma única
chamada a uma API externa.

## Stack

React 18 · Vite 5 · FastAPI · Uvicorn · Ollama (qwen2.5:3b) · pypdf ·
python-docx · Pillow + pytesseract · Docker Compose

## Arranque

Com o Docker Desktop a correr, não precisas de instalar Python, Node nem o
Ollama no sistema:

```powershell
Set-Content backend\chat_history.json "[]"   # só na primeira vez
docker compose up -d --build
docker compose exec ollama ollama pull qwen2.5:3b
```

A app fica em http://localhost:5173 e o backend em http://localhost:8000/api/health.
`local-docs/` e `backend/chat_history.json` são montados como volumes, por isso o
que a app escreve sobrevive a um `docker compose down`.

```powershell
docker compose logs -f backend    # logs em tempo real
docker compose down               # parar tudo
```

Sem Docker são três terminais — `ollama serve`, `uvicorn main:app --reload --port 8000`
em `backend/` (com o venv e o `requirements.txt` instalados), e `npm run dev` em
`frontend/`. O Ollama tem de arrancar **antes** do backend, senão o modelo aparece
como offline. Neste modo o OCR de imagens precisa do binário do Tesseract instalado
no sistema; sem ele cai para o modelo de visão do Ollama.

## Ecrãs

| Ecrã | O que faz |
| --- | --- |
| Chat | Resposta em streaming, Markdown, copiar, editar e reenviar, parar a meio, e o painel "a consultar: …" com os excertos que entraram no prompt |
| Documentos | Gerir `local-docs/` sem tocar no disco — criar pastas, carregar e apagar ficheiros |
| Arquitetura | Página dentro da app a explicar o RAG, o OCR e o pipeline |

A sidebar guarda as conversas recentes, alterna o tema e mostra o estado do
modelo (nome e se está disponível).

## API

| Endpoint | O que faz |
| --- | --- |
| `GET /api/health` | `{ollama_available, model}` |
| `POST /api/chat` | Resposta em SSE: um evento `scope`, depois `token` por cada pedaço, e um `done` com o id e o título da conversa |
| `GET /api/history` · `DELETE /api/history/{id}` | Conversas guardadas |
| `POST /api/upload` | Extrai texto de um ficheiro anexado a uma pergunta |
| `GET /api/docs` | Biblioteca local, por pasta |
| `POST /api/docs/folders` · `DELETE /api/docs/folders/{folder}` | Criar e apagar pastas |
| `POST /api/docs/{folder}/upload` · `DELETE /api/docs/{folder}/{filename}` | Ficheiros dentro de uma pasta |

CORS aceita `localhost` e `127.0.0.1` nas portas 5173–5175, porque o Vite avança
de porta quando encontra uma ocupada.

## Arquitetura

O React só desenha — no browser não há permissões para ler ficheiros locais nem
falar com o Ollama. Toda a lógica vive no backend:

```
React (:5173)  ->  FastAPI (:8000)  ->  Ollama (:11434)
```

```
backend/
  main.py             endpoints (chat SSE, histórico, upload, gestão de documentos)
  rag.py              deteção de desporto, indexação, scoring, system prompt, streaming
  docs_manager.py     pastas e ficheiros de local-docs/ a partir da UI
  file_extraction.py  PDF, DOCX e OCR de imagens para uploads do utilizador
  storage.py          histórico em chat_history.json
frontend/src/
  App.jsx             estado global e troca entre as três views
  api.js              fetch + parsing do streaming SSE
  index.css           tokens de tema (claro/escuro)
  components/         Sidebar, Welcome, ChatPanel, InputBar, Architecture, DocsLibrary
local-docs/           regulamentos por desporto, versionados no repo
docs/index.html       landing page do projeto
```

### Deteção de desporto

Antes de procurar, o backend passa a pergunta por listas de keywords em português
e inglês sobre o texto normalizado (Unicode NFKD, sem acentos). O desporto que
sai daí escolhe uma pasta de `local-docs/`, e só essa entra na busca.

A ordem das listas conta: `Football` (NFL) é testado **antes** de `Soccer`, porque
"futebol americano" contém "futebol" e o primeiro match ganha. Para pastas fora
da lista — como `hoquei-no-gelo`, criada pela UI — o scope força-se à mão
escrevendo `[folders: hoquei-no-gelo]` na própria mensagem.

### Retrieval

Não há vector store nem modelo de embeddings. Cada ficheiro é partido em blocos
de **900 caracteres com 150 de sobreposição**, tokenizado uma vez, e o índice
fica em cache 60 segundos — invalidada sempre que a app mexe nos documentos.

A relevância de cada bloco é uma soma ponderada de quatro sinais lexicais:

| Sinal | Peso |
| --- | --- |
| Similaridade léxica, `#(q ∩ c) / √(#q · #c)` | 0.65 |
| Cobertura da query | 0.15 |
| Bigramas em comum | 0.15 |
| Tokens do nome do ficheiro | 0.05 |
| Bónus de frase exata (aditivo) | +0.10 |

O trade-off é assumido: um scorer lexical falha sinónimos que embeddings
apanhariam. Para recuperar parte disso, a pergunta é traduzida PT→EN pelo próprio
Ollama e os tokens das duas línguas são unidos antes da busca — é o que permite
acertar em documentos ingleses a partir de perguntas em português.

Os 10 melhores blocos seguem para o prompt. Os excertos exatos vão também para o
frontend no evento `scope`, que é o que alimenta o "a consultar: …".

### Modo estrito

Com `LOCAL_RAG_STRICT_ONLY=true` (o default), o system prompt proíbe conhecimento
externo e fixa a frase de recusa: *"Não encontrei essa informação nos meus
ficheiros locais."* A regra está no backend, não fica à mercê de o modelo cumprir
a instrução.

### Ingestão

PDF via pypdf, DOCX via python-docx, e leitura direta para `.txt`, `.md`, `.csv`,
`.tsv`, `.json`, `.jsonl`, `.html`, `.xml`, `.yaml` e `.rtf`. Imagens tentam o
Tesseract primeiro e caem para o modelo de visão do Ollama se ele não estiver
instalado. Limite de 5 MB por ficheiro.

### Persistência

O histórico são as últimas 10 conversas em `backend/chat_history.json`, com o
título tirado dos primeiros 40 caracteres da primeira mensagem. A conversa é
gravada no `finally` do stream, por isso o texto já gerado fica guardado mesmo
que carregues em "Parar" ou a ligação caia. As sete pastas de desporto
incorporadas não podem ser apagadas pela UI — a deteção automática depende delas.

## Landing page

`docs/index.html` é um ficheiro único, estático, sem framework nem build step. O
mockup da interface é recriado em markup em vez de ser um screenshot, por isso
não fica desatualizado. O GitHub Pages serve a pasta `/docs` do `main`.

## Design

Sistema "Kinetic High-Performance": tokens definidos uma vez em
`frontend/src/index.css` e trocados por tema, azul `#0066ff` como acento e um
lime `#c3f400` reservado a sinais de estado. Claro e escuro com toggle na
sidebar, preferência guardada no browser — mas a sidebar mantém-se sempre escura
nos dois temas, como âncora visual fixa.
