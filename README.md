# Network Sentinel — Cadastro e Consulta de Hosts

Aplicação web com Flask + Pandas que usa arquivos Excel como banco de dados para gerenciamento de usuários de rede, com suporte a resolução de IP, execução de ping e histórico automático de consultas.

---

## 📁 Estrutura do Projeto

```
network-sentinel/
├── app.py                  ← Backend Flask (rotas, validações, lógica de negócio)
├── requirements.txt        ← Dependências Python
├── usuarios.xlsx           ← Gerado automaticamente ao rodar
├── historico.xlsx          ← Gerado automaticamente ao rodar
├── templates/
│   └── index.html          ← Estrutura HTML da interface
└── static/
    ├── css/
    │   └── style.css       ← Estilos visuais (Design System Network Sentinel)
    └── js/
        └── app.js          ← Lógica JavaScript do frontend
```

---

## ⚙️ Pré-requisitos

- Python 3.8 ou superior
- pip

---

## 🚀 Como Rodar

### 1. Clone ou baixe os arquivos do projeto

Mantenha a estrutura de pastas exatamente como descrita acima.

### 2. Crie um ambiente virtual (recomendado)

```bash
cd network-sentinel
python -m venv venv
```

Ative o ambiente virtual:

- **Windows:**
  ```bash
  venv\Scripts\activate
  ```
- **macOS/Linux:**
  ```bash
  source venv/bin/activate
  ```

### 3. Instale as dependências

```bash
pip install -r requirements.txt
```

### 4. Rode a aplicação

```bash
python app.py
```

### 5. Acesse no navegador

Abra: [http://localhost:5000](http://localhost:5000)

---

## 📌 Rotas da API

| Rota          | Método   | Descrição                                          |
|---------------|----------|----------------------------------------------------|
| `/`           | `GET`    | Página principal                                   |
| `/cadastrar`  | `POST`   | Cadastra novo usuário                              |
| `/buscar`     | `GET`    | Busca por Nome, RACF ou Hostname                   |
| `/editar`     | `PUT`    | Edita registro existente                           |
| `/excluir`    | `DELETE` | Remove usuário por RACF                            |
| `/ping`       | `POST`   | Executa ping e registra no histórico               |
| `/historico`  | `GET`    | Retorna os pings válidos (não expirados)           |

---

## 🗂️ Modelo de Dados

### `usuarios.xlsx` — Cadastro de usuários

| Coluna      | Tipo   | Descrição                          |
|-------------|--------|------------------------------------|
| `Nome`      | Texto  | Nome completo do usuário           |
| `RACF`      | Texto  | Identificador único (máx. 7 chars) |
| `Funcional` | Texto  | Código numérico (máx. 9 dígitos)   |
| `Hostname`  | Texto  | Hostname único da máquina          |

### `historico.xlsx` — Histórico de pings

| Coluna           | Tipo   | Descrição                           |
|------------------|--------|-------------------------------------|
| `data_hora`      | Texto  | Data/hora no formato DD/MM/YYYY HH:MM:SS |
| `nome`           | Texto  | Nome do usuário ou hostname         |
| `hostname`       | Texto  | Hostname consultado                 |
| `ip`             | Texto  | IP resolvido                        |
| `status`         | Texto  | Online / Offline / Host não encontrado |
| `tempo_resposta` | Texto  | Latência medida (ex: `12 ms`)       |

---

## 🔒 Regras de Validação

### Campo RACF
- **Obrigatório** em cadastro e edição.
- **Máximo de 7 caracteres** — valores acima são rejeitados no frontend (via `maxlength`) e no backend (HTTP 400).
- **Único** — não é possível cadastrar ou editar com uma RACF já existente (validação case-insensitive).

### Campo Funcional
- **Obrigatório** em cadastro e edição.
- **Apenas números** — caracteres não numéricos são bloqueados no frontend (`oninput`) e rejeitados no backend via regex `\d{1,9}`.
- **Máximo de 9 dígitos** — valores acima são rejeitados no frontend (via `maxlength`) e no backend (HTTP 400).

### Campo Hostname
- **Obrigatório** em cadastro e edição.
- **Único** — não é possível vincular o mesmo hostname a mais de um usuário (validação case-insensitive).
- Na rota `/ping`, o hostname passa por validação de caracteres permitidos (`^[a-zA-Z0-9.\-_]+$`).

### Validações gerais
- Todos os campos são obrigatórios em cadastro e edição.
- Duplicatas de RACF e Hostname são verificadas tanto no cadastro quanto na edição.
- No modo edição, o registro atual é excluído da verificação de duplicatas para permitir salvar sem alteração nos campos únicos.

---

## 🕐 Expiração Automática do Histórico

O histórico de pings é **automaticamente limpo** a cada consulta à rota `/historico` ou a cada novo ping registrado:

- Registros com **mais de 10 minutos** são identificados e excluídos do arquivo `historico.xlsx`.
- A expiração é baseada no campo `data_hora` de cada registro (formato `DD/MM/YYYY HH:MM:SS`).
- Registros com formato de data inválido são preservados (comportamento seguro).
- O arquivo é regravado somente quando há remoções efetivas, evitando I/O desnecessário.
- Não há limite fixo de quantidade de registros; o controle é feito exclusivamente por tempo.

**Função responsável:** `purgar_historico_expirado(df)` em `app.py`.

---

## 🔍 Resolução de IP e Ping

A rota `/ping` realiza a resolução do hostname em duas etapas antes de executar o ping:

1. **DNS/IPv4** via `socket.getaddrinfo` forçando `AF_INET` (IPv4 puro).
2. **NetBIOS/Windows** via `nmblookup` (fallback para hostnames do tipo `DESKTOP-XXXXX`). Requer o pacote `samba-common` ou `winbind` no Linux.

Caso nenhuma resolução tenha êxito, o ping é tentado diretamente contra o hostname informado.

**Extração de latência:** O RTT é extraído da saída do comando `ping` via regex `[<=](\d+\.?\d*)\s*ms`. Caso não seja encontrado, o tempo é calculado dividindo a duração total da execução por 4 pacotes.

---

## 📝 Observações

- Os arquivos `usuarios.xlsx` e `historico.xlsx` são criados automaticamente na primeira execução caso não existam.
- A busca é **case-insensitive** e suporta correspondência parcial nos campos Nome, RACF e Hostname.
- Para produção, remova `debug=True` em `app.py` e considere um servidor WSGI como Gunicorn.
- O campo `Funcional` é exibido nos cards de resultado da aba **Hosts**.
