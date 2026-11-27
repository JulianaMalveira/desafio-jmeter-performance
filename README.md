# Desafio Prático – Testes de Performance com Apache JMeter

Este repositório contém a suíte completa de automação de testes de performance desenvolvida em **Apache JMeter** para a **API de Pets**, criada como trabalho prático da disciplina de Automação de Testes de Performance do curso de Pós-Graduação em Automação de Testes de Software.

O objetivo deste projeto é avaliar o comportamento, a resiliência e a vazão da aplicação em dois modelos distintos de carga (**Modelo Fechado** e **Modelo Aberto**), garantindo o fluxo completo de autenticação via token JWT e a execução validada dos endpoints de cadastro e consulta de pets (`/pets`).

---

## 📐 Arquitetura da Aplicação e Endpoints

A aplicação testada é a **API de Pets**, uma API REST desenvolvida em Node.js/Express com armazenamento temporário em memória RAM e documentação interativa via Swagger UI.

- **Repositório da API:** [tiagotaq/api-de-pets](https://github.com/tiagotaq/api-de-pets)
- **Ambiente de Execução:** `http://localhost:3000`
- **Documentação Swagger:** `http://localhost:3000/api-docs`

### Matriz de Endpoints Cobertos no Teste

| Endpoint | Método HTTP | Autenticação Requerida | Descrição do Fluxo |
| :--- | :--- | :--- | :--- |
| `/login` | `POST` | Não | Autenticação do usuário e geração do token JWT de acesso |
| `/pets` | `POST` | Sim (`Bearer Token`) | Cadastro de novo pet na base de dados com nome dinâmico |
| `/pets` | `GET` | Sim (`Bearer Token`) | Consulta e listagem dos pets cadastrados |

---

## 🛠️ Estrutura do Plano de Testes JMeter (`.jmx`)

O plano de testes foi construído no arquivo [`desafio_pets.jmx`](desafio_pets.jmx) mantendo uma arquitetura modular, parametrizada e com URL/porta centralizadas.

```text
Desafio Performance - API Pets (desafio_pets.jmx)
 ├── HTTP Request Defaults (http://localhost:3000)
 ├── setUp Thread Group (1 thread, 1 loop — execução única de autenticação)
 │    └── POST /login
 │         ├── HTTP Header Manager (Content-Type: application/json)
 │         ├── JSON Extractor (Variável: tokenExtraido via $.token)
 │         └── JSR223 PostProcessor (Groovy: grava token_prop)
 ├── Thread Group - Modelo Fechado (5 threads, ramp-up 5s, 2 loops)
 │    ├── HTTP Header Manager (Authorization: Bearer ${__P(token_prop)} + Content-Type)
 │    ├── POST /pets (Payload dinâmico com timestamp + número aleatório)
 │    ├── GET /pets
 │    └── Uniform Random Timer (Think Time: 500 ms + até 1000 ms)
 ├── Thread Group - Modelo Aberto (OpenModelThreadGroup: rate(2/s) random_arrivals(10 s) rate(2/s))
 │    ├── HTTP Header Manager (Authorization: Bearer ${__P(token_prop)} + Content-Type)
 │    ├── POST /pets (Payload dinâmico com timestamp + número aleatório)
 │    └── GET /pets
 └── View Results Tree (desabilitado por padrão para execução de carga)
```

---

## 🔐 Gestão de Autenticação e Propriedades Globais

As rotas `/pets` exigem a passagem de um token de autorização no formato `Bearer Token`. Como as variáveis em tempo de execução no JMeter possuem escopo limitado ao próprio Thread Group, foi implementado o padrão de propagação via propriedades globais:

1. **`setUp Thread Group`**: Executado prioritariamente antes dos demais grupos para realizar a chamada `POST /login` com o payload:
   ```json
   {"username": "tiago", "password": "123456"}
   ```
2. **`JSON Extractor`**: Extrai o campo `token` da resposta JSON através da expressão `$.token` e armazena na variável local `tokenExtraido`.
3. **`JSR223 PostProcessor` (Groovy)**: Converte a variável local em uma propriedade global do JMeter acessível por qualquer Thread Group do plano:
   ```groovy
   props.put('token_prop', vars.get('tokenExtraido'))
   ```
4. **Reutilização nos Grupos de Carga**: Nos grupos de carga fechado e aberto, o token é injetado via `HTTP Header Manager`:
   - **Name:** `Authorization`
   - **Value:** `Bearer ${__P(token_prop)}`

---

## 🎲 Prevenção de Colisão e Dados Únicos

Na `api-de-pets`, a tentativa de cadastrar dois pets com o mesmo nome para o mesmo usuário retorna `409 Conflict`. Para evitar falhas por duplicidade durante execuções concorrentes, o payload de `POST /pets` combina as funções nativas `${__time()}` e `${__Random(1000,9999)}`:

```json
{
  "name": "Rex_${__time()}_${__Random(1000,9999)}",
  "type": "Cão"
}
```

---

## 📊 Comparativo dos Modelos de Carga Implementados

A suíte contempla os dois principais modelos de carga utilizados na engenharia de performance:

### 1. Modelo Fechado (Closed Workload Model)
- **Conceito:** O número de usuários virtuais concorrentes é fixo (`5` threads, ramp-up de `5` segundos, `2` iterações). Cada usuário envia uma requisição, aguarda a resposta do servidor e só então avança para a próxima.
- **Comportamento do Throughput:** A vazão de requisições por segundo (RPS) depende diretamente do tempo de resposta da API. Se o servidor apresentar degradação, a vazão cai proporcionalmente.
- **Think Time:** Inclui o componente `Uniform Random Timer` (`500` ms fixo + até `1000` ms aleatório) para simular o tempo de pausa entre ações reais.

### 2. Modelo Aberto (Open Workload Model)
- **Conceito:** Implementado nativamente com **`OpenModelThreadGroup`**, injetando chegadas a uma taxa constante (`rate(2/s) random_arrivals(10 s) rate(2/s)`), independentemente da latência do servidor.
- **Comportamento do Throughput:** A taxa de chegada é mantida conforme o agendamento configurado mesmo quando a aplicação apresenta maior tempo de resposta.
- **Regra de Execução:** Ausência total de *Think Time* (nenhum Timer configurado), garantindo que a taxa de chegada estipulada não seja distorcida por pausas artificiais.

---

## 🚀 Guia de Execução do Projeto

### Pré-requisitos
- **Java Development Kit (JDK 8 ou JDK 21)** instalado e configurado na variável de ambiente `JAVA_HOME`.
- **Node.js (v18+)** instalado para execução da API local.
- **Apache JMeter 5.6.3** (ou 5.5+ com suporte ao `OpenModelThreadGroup`).

### Passo 1: Inicializar a API de Pets
```powershell
git clone https://github.com/tiagotaq/api-de-pets
cd api-de-pets
npm install
npm start
```
Confirme que a API está respondendo em `http://localhost:3000/api-docs`.

### Passo 2: Validar o Plano de Testes na Interface Gráfica (Smoke Test)
1. Abra o Apache JMeter (`jmeter.bat` no Windows ou `./jmeter.sh` no Linux/Mac).
2. Abra o arquivo `desafio_pets.jmx`.
3. Caso queira inspecionar os resultados visualmente, habilite temporariamente o listener **View Results Tree** (**botão direito → Enable**).
4. Clique em **Play** (ícone verde) e verifique se `POST /login`, `POST /pets` e `GET /pets` retornam status `200`/`201` com `0` erros.
5. Antes de salvar para testes de carga, desabilite novamente o **View Results Tree** (**botão direito → Disable**) para economizar memória.

### Passo 3: Execução de Carga em Modo Linha de Comando (Non-GUI / CLI)
Para evitar sobrecarga de memória da interface gráfica durante testes de carga, execute via terminal:

```powershell
jmeter -n -t desafio_pets.jmx -l resultados.jtl -e -o ./dashboard_relatorio
```

**Parâmetros do comando:**
- `-n`: Executa o JMeter em modo Non-GUI (linha de comando).
- `-t`: Caminho para o arquivo do plano de testes (`.jmx`).
- `-l`: Caminho para gravação do arquivo de resultados (`.jtl`).
- `-e`: Gera o relatório HTML Dashboard ao final da execução.
- `-o`: Pasta de destino do painel HTML *(a pasta deve estar vazia ou não existir antes da execução)*.

---

## 📈 Resultados e Dashboard HTML

Ao finalizar a execução via CLI, abra o arquivo `./dashboard_relatorio/index.html` no navegador para analisar:
- **Summary Report:** Métricas de Apdex, percentis (`p90`, `p95`, `p99`), média, mediana e throughput (RPS).
- **Charts Overview:** Gráficos de tempo de resposta ao longo do tempo, distribuição de erros e *Throughput vs. Active Threads*.

---

## 👩‍💻 Autora

Projeto desenvolvido por **Juliana Malveira** como trabalho prático para a disciplina de **Automação de Testes de Performance** (Pós-Graduação em Automação de Testes de Software).

**Ferramentas utilizadas:** Apache JMeter 5.6.3, Node.js, Express, Groovy Scripting, Git.
