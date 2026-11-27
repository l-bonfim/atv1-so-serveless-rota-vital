# Atividade individual 1 – Serverless (Rota Vital)
 
Infraestrutura de Software · ADS · CESAR School · 2026.2
 
Função serverless que gera a **guia de transporte** de uma requisição do Rota Vital, acionada por evento (`sam local invoke`) e por HTTP (`sam local start-api`).
 
## Caminho escolhido
 
**Caminho A – AWS SAM CLI local**, sem conta na AWS.
 
## Estrutura do repositório
 
```
.
├── README.md
├── evidencias/                 # prints dos comandos e saídas (01 a 17)
└── guia-transporte/            # projeto criado com sam init
    ├── template.yaml           # rota POST /guias, Timeout: 10 (estado final)
    ├── events/pedido.json      # evento API Gateway usado no sam local invoke
    └── hello_world/app.py      # handler do enunciado (sem alterações)
```
 
## Como rodar
 
### Ambiente usado
 
- Pop!_OS 22.04 (x86_64)
- Docker 29.10.0 (usuário no grupo `docker`, sem sudo)
- AWS SAM CLI 1.174.1
- Python local 3.10. Por isso o build usa `sam build --use-container`, que compila com a imagem oficial de Python 3.12. Com Python 3.12 instalado, `sam build` funciona igual.
### Passo a passo
 
```bash
cd guia-transporte
 
# build
sam build --use-container
 
# Passo 3 – invocação com evento
sam local invoke -e events/pedido.json
 
# Passo 4 – gatilho HTTP (terminal 1)
sam local start-api
# ou, com reaproveitamento explícito:
sam local start-api --warm-containers LAZY
 
# Passo 4 – chamadas (terminal 2, três vezes)
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
    -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
 
# Passo 5 – limite de tempo
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
    -d '{"requisicaoId":"R-003","demorar":true}'
```
 
Para reproduzir o erro de tempo limite, deixe `Timeout: 3` em `Globals` no `template.yaml`, rode `sam build --use-container` e reinicie o `start-api`. Para o ajuste, troque para `Timeout: 10` e repita o build.
 
### Observações do experimento
 
- **Evento com body em base64:** nesta versão do SAM, o `sam local generate-event apigateway aws-proxy --body ...` gerou o `body` codificado em base64, com `"isBase64Encoded": true`. Como o handler faz `json.loads` direto no body, a primeira invocação falhou com `JSONDecodeError` (`evidencias/04`). Decodifiquei o body do `events/pedido.json` e marquei `isBase64Encoded: false` (`evidencias/05`). O `app.py` continua idêntico ao do enunciado.
- **Reaproveitamento por padrão:** no SAM 1.174.1, o `sam local start-api` já reaproveitou o container mesmo sem `--warm-containers` (log: `Lambda function 'HelloWorldFunction' is already running`). Por isso o contador subiu nos dois modos.
- O `template.yaml` do repositório está com `Timeout: 10`, que é o estado final depois do Passo 5.
## Resultados registrados
 
### Passo 3 – `sam local invoke` (cada invocação cria um container novo)
 
| Invocação | Resultado | Duration | Tempo total (`time`) | `invocacaoNesteAmbiente` |
|---|---|---|---|---|
| 1ª | timeout de 3 s (o runtime não ficou pronto a tempo) | – | 16,6 s | – |
| 2ª | 201 | 221,60 ms | 12,9 s | 1 |
| 3ª | 201 | 123,01 ms | 12,7 s | 1 |
 
### Passo 4 – `sam local start-api`
 
| Modo | Chamada | `invocacaoNesteAmbiente` | Init Duration | Duration |
|---|---|---|---|---|
| sem flag | 1 | 1 | 37,82 ms | 2279,98 ms |
| sem flag | 2 | 2 | – | 3,81 ms |
| sem flag | 3 | 3 | – | 2,29 ms |
| `--warm-containers LAZY` | 1 | 1 | 0,04 ms | 99,68 ms |
| `--warm-containers LAZY` | 2 | 2 | – | 3,15 ms |
| `--warm-containers LAZY` | 3 | 3 | – | 1,71 ms |
 
### Passo 5 – limite de tempo (`demorar: true`, a função dorme 5 s)
 
| Timeout | Resultado | Duration | Billed Duration |
|---|---|---|---|
| 3 s | `timed out after 3 seconds`, HTTP 502 `Internal server error` | 3000,00 ms | 3000 ms |
| 10 s | HTTP 201, `"guia": "G-R-003"` | 5153,13 ms | 5154 ms |
 
## Respostas
 
### 1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?
 
A primeira invocação pega o ambiente "frio" (cold start). Antes de executar o handler, a plataforma precisa passar pela fase de **Init**: criar o ambiente de execução (aqui, um container Docker), carregar o runtime do Python 3.12, montar o código da função e rodar tudo que está fora do handler (os `import` e a criação da variável global `contador`). Só depois disso o handler é chamado.
 
Na segunda invocação com o ambiente reaproveitado, a fase de Init não acontece de novo: o handler roda direto. No `start-api`, a primeira chamada levou 2279,98 ms e as seguintes, de 2 a 4 ms, e o `Init Duration` só apareceu na primeira. No `sam local invoke`, a primeira execução nem chegou a rodar o handler: o container demorou mais que os 3 s do Timeout para ficar pronto e a função estourou o limite. As seguintes responderam em 120 a 220 ms. Localmente o cold start é exagerado, porque inclui subir um container Docker na máquina.
 
### 2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global da função?
 
No `sam local invoke`, o valor foi sempre **1**, porque cada execução cria um container novo e a variável começa do zero.
 
No `start-api`, o valor foi **1, 2, 3** tanto sem flag quanto com `--warm-containers LAZY`. Nesta versão do SAM (1.174.1), o `start-api` já reaproveita o container por padrão. Com LAZY, o container é criado na primeira chamada e mantido até o servidor parar. Quando reiniciei o servidor, o contador **voltou para 1**.
 
Isso mostra que uma variável global só "lembra" de algo enquanto o mesmo ambiente estiver vivo, e a plataforma decide quando criar, reaproveitar ou destruir esse ambiente. No Lambda real, várias chamadas simultâneas rodam em ambientes diferentes, cada um com o seu contador. Então não dá para guardar estado da aplicação numa variável global. Ela serve só como cache (por exemplo, uma conexão com o banco reaproveitada entre chamadas). Dados que precisam persistir vão para um serviço externo, como banco de dados, DynamoDB, S3 ou Redis.
 
### 3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?
 
Com `Timeout: 3`, a função dormiu 5 s e foi interrompida no meio: o log mostrou `Function 'HelloWorldFunction' timed out after 3 seconds`, `Duration: 3000.00 ms` e o cliente recebeu **HTTP 502** (`Internal server error`). A função nunca chegou ao `return`, mas os 3000 ms foram contados como tempo faturado. Depois de aumentar para `Timeout: 10` e refazer o build, a mesma chamada respondeu **201** com `Duration: 5153.13 ms`.
 
O tempo máximo de uma execução no Lambda é **15 minutos (900 segundos)**. Para trabalhos mais longos, a saída é não fazer tudo numa função só:
- quebrar o trabalho em partes menores, encadeadas por uma fila (SQS) ou orquestradas com **Step Functions**;
- responder logo ao cliente (por exemplo, HTTP 202) e processar em segundo plano, avisando no final;
- usar um serviço sem esse limite, como contêineres (ECS/Fargate) ou AWS Batch.
### 4. Quanto custa um milhão de execuções desta função com 128 MB e 100 ms?
 
Preços de us-east-1: US$ 0,20 por milhão de requisições e US$ 0,0000166667 por GB-s.
 
**Requisições:**
1.000.000 × (US$ 0,20 / 1.000.000) = **US$ 0,20**
 
**Computação (GB-s):**
- memória: 128 MB = 128 / 1024 = 0,125 GB
- por execução: 0,125 GB × 0,1 s = 0,0125 GB-s
- total: 0,0125 × 1.000.000 = 12.500 GB-s
- custo: 12.500 × US$ 0,0000166667 = **US$ 0,2083**
**Total:** US$ 0,20 + US$ 0,2083 ≈ **US$ 0,41 por milhão de execuções**
 
Essa conta ignora o plano gratuito. Na faixa sempre gratuita (1 milhão de requisições e 400 mil GB-s por mês), esse volume sairia por US$ 0,00.
 
### 5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?
 
**Faria sentido** em tarefas curtas, sem estado e disparadas por evento, como esta: gerar a guia de transporte quando uma requisição hospitalar é aprovada, gerar o PDF ou etiqueta da guia, enviar notificações (e-mail ou SMS para o hospital quando a bolsa sair) e validar os dados de uma requisição. Essas tarefas têm volume irregular e duram milissegundos. Pagar só por execução é barato, e a escala é automática se chegarem muitas requisições de uma vez.
 
**Não faria sentido** no controle do estoque de bolsas em si. Ele exige consistência forte (duas requisições não podem reservar a mesma bolsa) e estado guardado com segurança, então não pode depender de variável global. Também não faria sentido para processos longos ou contínuos, como o acompanhamento em tempo real do transporte ou relatórios pesados que passem de 15 minutos. Nesses casos, um serviço sempre ativo com banco de dados transacional funciona melhor, e o cold start também atrapalharia uma tela que precisa responder rápido o tempo todo.
 
## Evidências
 
Prints na pasta [`evidencias/`](evidencias/):
 
| Arquivo | O que mostra |
|---|---|
| `01-environment.png` | Versões do Docker e do SAM CLI |
| `02-sam-init.png` | Projeto criado com `sam init` |
| `03-build-container.png` | `sam build --use-container` com sucesso |
| `04-invoke-error-base64.png` | Invocação falhando por causa do body em base64 |
| `05-fixing-error.png` | Correção do body do evento |
| `06-invoke-1.png` | 1ª invocação: timeout no cold start |
| `07-invoke-2.png` | 2ª invocação: 201 |
| `08-invoke-3.png` | 3ª invocação: 201 |
| `09` a `11-start-api-call-*.png` | `start-api` sem flag: contador 1, 2, 3 |
| `12` a `14-warm-lazy-call-*.png` | `start-api --warm-containers LAZY`: contador 1, 2, 3 |
| `15-timeout-call-3s.png` | Erro de tempo limite com Timeout 3 s |
| `16-timeout-10s-build.png` | Timeout alterado para 10 s e novo build |
| `17-timeout-10s-success.png` | Chamada com `demorar` concluída com Timeout 10 s |
 
Vídeo (até 3 min): _adicionar link_
 
## Uso de IA
 
Declaração conforme a PO-CESAR-021.
 
- **Ferramenta:** Claude (Anthropic).
- **Finalidade:** suporte durante a atividade e correção de erros. A IA orientou a instalação do Docker e do SAM CLI no Pop!_OS, explicou e ajudou a corrigir o erro `JSONDecodeError` do evento com body em base64, e sugeriu o uso de `sam build --use-container` por causa do Python 3.10 local. Também ajudou a organizar e redigir este README a partir dos resultados que obtive, incluindo o rascunho das respostas.
- **Como validei:** executei todos os comandos na minha máquina e conferi as saídas (prints em `evidencias/`). Conferi a correção do evento com `grep` antes e depois. Revisei as respostas comparando com os números que obtive e com o conteúdo da Aula 007, e refiz a conta de custo.
 