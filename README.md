# Desafio Performance - API Pets (JMeter)

Plano de testes de performance em **Apache JMeter 5.6.3** para a [API de Pets](https://github.com/tiagotaq/api-de-pets), contemplando autenticação JWT compartilhada entre Thread Groups, Modelo Fechado e Modelo Aberto.

## Estrutura do Plano (`desafio_pets.jmx`)

1. **HTTP Request Defaults**
   - Protocolo: `http` | Servidor: `localhost` | Porta: `3000`
2. **setUp Thread Group** (`1` thread, `1` loop)
   - `POST /login` com credenciais válidas (`username` / `password`)
   - **JSON Extractor** (`$.token` → `tokenExtraido`)
   - **JSR223 PostProcessor (Groovy)**: `props.put('token_prop', vars.get('tokenExtraido'))`
3. **Thread Group - Modelo Fechado** (`5` threads, ramp-up `5`s, `2` loops)
   - **HTTP Header Manager**: `Authorization: Bearer ${__P(token_prop)}` e `Content-Type: application/json`
   - `POST /pets` (com nome único usando `${__time()}` e `${__Random(1000,9999)}`)
   - `GET /pets`
   - **Uniform Random Timer** (*Think Time*: `500` ms + até `1000` ms)
4. **Thread Group - Modelo Aberto** (`OpenModelThreadGroup`)
   - Schedule: `rate(2/s) random_arrivals(10 s) rate(2/s)`
   - **HTTP Header Manager**: `Authorization: Bearer ${__P(token_prop)}` e `Content-Type: application/json`
   - `POST /pets` e `GET /pets`
   - Sem timers de pausa (*Think Time*), preservando a taxa de chegada
5. **View Results Tree**
   - Desabilitado por padrão para execução de carga com menor consumo de memória

## Como Executar

1. Inicie a API de Pets em `http://localhost:3000`:
   ```powershell
   git clone https://github.com/tiagotaq/api-de-pets
   cd api-de-pets
   npm install
   npm start
   ```
2. Abra `desafio_pets.jmx` no **Apache JMeter** (5.5+ / 5.6.3) e execute o plano de testes.
