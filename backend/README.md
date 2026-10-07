# Backend — relatório de debug

Método: `docker compose up --build` → ler logs → isolar a camada → corrigir
um erro por vez → subir de novo. Cada correção virou um commit
`fix: <hipótese> → <correção>`.

## Alvos da variante

| Parâmetro | Valor |
| --- | --- |
| Porta publicada | `9102` |
| Banco | `prova_55` |
| Origem do CORS | `http://localhost:3000` |

## Hipótese → evidência → correção

| # | Camada | Hipótese | Evidência (log) | Correção |
| --- | --- | --- | --- | --- |
| 1 | Compilação | `COPY --from` aponta para estágio que não existe | `pull access denied ... docker.io/library/builder:latest (did you mean build?)` | `--from=build`, igual ao `AS build` |
| 2 | Compilação | `npm ci` exige lockfile e o frontend não tem[^lock] | falha em `RUN npm ci` | `RUN npm install` |
| 3 | Compilação | versão do driver PostgreSQL inexistente | `Could not find artifact org.postgresql:postgresql:jar:42.7.999` | versão `42.7.3` |
| 4 | Compilação | typo no import do JPA | `package jakarta.persistense does not exist` (`Produto.java:[3,27]`) | `jakarta.persistence` |
| 5 | Compilação | typo na anotação | `cannot find symbol: class GetMaping` (`ProdutoController.java:[29,6]`) | `@GetMapping` |
| 6 | Configuração | healthcheck usa usuário inexistente | `pg_isready -U admin`, e o usuário do banco é `prova` | `-U prova` |
| 7 | Configuração | banco e portas fora da variante | `POSTGRES_DB: prova`, `server.port=8081`, `8080:8080` | `prova_55` nos dois arquivos, `server.port=8080`, `9102:8080` |
| 8 | Startup | `index.html` fora de `public/` | `Could not find a required file. Name: index.html` | mover para `frontend/public/` |
| 9 | Startup | arquivo de entrada com nome que o `react-scripts` não procura | `Could not find a required file. Name: index.js` | `main.jsx` → `index.jsx` |
| 10 | Lógica | frontend chama porta e caminho errados | `API = 'http://localhost:8080/produto'` | `http://localhost:9102/produtos` |
| 11 | Lógica | CORS permite a origem errada[^cors] | `403 Invalid CORS request` com `Origin: http://localhost:3000` | `@CrossOrigin(origins = "http://localhost:3000")` |

## Hipóteses descartadas

> [!WARNING]
> **`POST /produto` com `{"ok": true}`:** o exemplo do enunciado contradiz o
> contrato. O contrato manda `/produtos` e resposta `201`, então segui o contrato.

> [!NOTE]
> **Porta 9102 ocupada:** `docker compose up` falhou com
> `bind: address already in use`. Não era bug do projeto: um processo local
> meu usava a porta, e o `docker ps` mostrou que nenhum container a ocupava.
> Encerrei o processo e não alterei a porta do compose.

> [!NOTE]
> **`FATAL: database "prova" does not exist` no log do db:** vem do
> `pg_isready` do healthcheck, que consultava o banco com o nome do usuário. Não
> impedia o `db` de ficar `healthy`.

## Verificação final

```
curl -i http://localhost:9102/produtos                      # 200 []
curl -i -X POST http://localhost:9102/produtos \
  -H "Content-Type: application/json" \
  -d '{"nome":"Teclado","precoCentavos":15000,"quantidade":7}'  # 201
curl -i -H "Origin: http://localhost:3000" \
  http://localhost:9102/produtos                            # CORS ok
```

[^lock]: O `npm ci` só funciona com `package-lock.json`. Em vez de gerar um
lockfile, troquei por `npm install`, que é a mudança mínima.
[^cors]: O browser bloqueia chamadas entre origens quando o servidor não
devolve `Access-Control-Allow-Origin` compatível. O `curl` sem `Origin`
funcionava, e só o teste com o header `Origin` revelou o erro.