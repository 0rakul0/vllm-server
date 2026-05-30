# Servidor vLLM local com Docker

Este projeto sobe um servidor **vLLM** em Docker para executar um modelo do Hugging Face localmente, expondo uma API compatível com o padrão da OpenAI.

No estado atual, o servidor roda o modelo:

```text
Qwen/Qwen2.5-1.5B-Instruct
```

na porta local:

```text
http://localhost:8008
```

\---

## 1\. Estrutura esperada

```text
E:/vllm-server/
├── docker-compose.yml
├── .env
└── models/
```

O diretório `models/` é usado para armazenar os modelos baixados do Hugging Face.

\---

## 2\. Arquivo `.env`

Crie um arquivo chamado `.env` na mesma pasta do `docker-compose.yml`:

```env
HF\_TOKEN=hf\_seu\_token\_aqui
```

O token é usado para baixar modelos do Hugging Face. Para modelos públicos, ele pode não ser obrigatório, mas é útil manter configurado. Para modelos restritos, como alguns modelos da Meta/Llama, o token precisa ter permissão de leitura e a conta precisa ter aceitado os termos do modelo.

```gitignore
.env
models/
```

\---

## 3\. `docker-compose.yml` atual

```yaml
services:
  vllm:
    image: vllm/vllm-openai:latest
    container\_name: vllm-openai
    ipc: host

    command: >
      --model Qwen/Qwen2.5-1.5B-Instruct
      --download-dir /models
      --host 0.0.0.0
      --port 8008
      --gpu-memory-utilization 0.50
      --max-model-len 1024
      --max-num-seqs 1
      --enforce-eager

    shm\_size: "8gb"

    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: \[gpu]

    volumes:
      - E:/vllm-server/models:/models

    ports:
      - "8008:8008"

    environment:
      HF\_TOKEN: ${HF\_TOKEN}
```

\---

## 4\. O que faz cada parâmetro

### `services`

Define os serviços que o Docker Compose vai subir. Neste caso, existe apenas um serviço chamado `vllm`.

\---

### `vllm`

Nome lógico do serviço dentro do Docker Compose.

Você pode iniciar, parar e consultar logs desse serviço com comandos como:

```bash
docker compose up -d
```

```bash
docker logs -f vllm-openai
```

\---

### `image: vllm/vllm-openai:latest`

Define a imagem Docker usada.

A imagem `vllm/vllm-openai` é a imagem oficial do vLLM para subir um servidor compatível com a API da OpenAI.

Ela **não baixa modelos da OpenAI** e **não usa a plataforma da OpenAI**. O termo `openai` significa apenas que o servidor expõe endpoints compatíveis, como:

```text
/v1/models
/v1/chat/completions
/v1/completions
```

\---

### `container\_name: vllm-openai`

Define o nome fixo do container.

Isso facilita consultar os logs:

```bash
docker logs -f vllm-openai
```

Sem esse campo, o Docker Compose geraria um nome automático.

\---

### `ipc: host`

Compartilha o namespace de IPC do host com o container.

É comum em workloads de machine learning porque bibliotecas como PyTorch/vLLM podem precisar de memória compartilhada para operações internas. A própria documentação do vLLM mostra exemplos Docker usando `--ipc=host`.

\---

### `command`

Define os argumentos usados para iniciar o servidor vLLM dentro do container.

No seu caso:

```yaml
command: >
  --model Qwen/Qwen2.5-1.5B-Instruct
  --download-dir /models
  --host 0.0.0.0
  --port 8008
  --gpu-memory-utilization 0.50
  --max-model-len 1024
  --max-num-seqs 1
  --enforce-eager
```

\---

### `--model Qwen/Qwen2.5-1.5B-Instruct`

Define o modelo que o vLLM vai carregar.

Neste caso, o modelo vem do Hugging Face:

```text
Qwen/Qwen2.5-1.5B-Instruct
```

Ao iniciar, o vLLM baixa o modelo, se ele ainda não existir no cache local, e depois carrega os pesos para execução.

Observação: versões recentes do vLLM avisam que o modelo deve ser passado preferencialmente como argumento posicional, em vez de `--model`. Portanto, no futuro, este formato pode ser preferível:

```yaml
command: >
  Qwen/Qwen2.5-1.5B-Instruct
  --download-dir /models
  --host 0.0.0.0
  --port 8008
```

\---

### `--download-dir /models`

Define onde o vLLM/Hugging Face vai armazenar os arquivos baixados do modelo dentro do container.

Como existe este volume:

```yaml
volumes:
  - E:/vllm-server/models:/models
```

os modelos baixados dentro do container em `/models` ficam salvos no Windows em:

```text
E:\\vllm-server\\models
```

Isso evita baixar o modelo novamente toda vez que o container for recriado.

\---

### `--host 0.0.0.0`

Faz o servidor escutar em todas as interfaces de rede dentro do container.

Isso é necessário para que a porta publicada pelo Docker fique acessível a partir do Windows pelo endereço:

```text
http://localhost:8008
```

\---

### `--port 8008`

Define a porta interna usada pelo servidor vLLM.

No compose, ela é combinada com:

```yaml
ports:
  - "8008:8008"
```

O primeiro `8008` é a porta do Windows/host.

O segundo `8008` é a porta dentro do container.

\---

### `--gpu-memory-utilization 0.50`

Controla a fração da memória da GPU que o vLLM tentará usar.

No seu caso:

```yaml
--gpu-memory-utilization 0.50
```

significa que o vLLM tentará trabalhar com cerca de 50% da VRAM disponível para essa instância.

Em uma RTX 3050 de 8 GB, isso é uma configuração conservadora e ajuda a evitar erros de falta de memória.

Exemplos:

```text
0.50 = usa até cerca de 50% da VRAM
0.75 = usa até cerca de 75% da VRAM
0.80 = usa até cerca de 80% da VRAM
0.90 = usa até cerca de 90% da VRAM
```

Quanto maior o valor, mais espaço o vLLM tem para modelo e KV cache, mas maior a chance de erro se já houver outros processos usando GPU.

\---

### `--max-model-len 1024`

Define o tamanho máximo de contexto do modelo, em tokens.

Esse limite inclui entrada e saída.

Exemplo aproximado:

```text
prompt com 700 tokens + resposta com 300 tokens = 1000 tokens
```

Quanto maior esse valor, mais memória o vLLM precisa reservar para o KV cache.

Para GPU com 8 GB de VRAM, valores como `512`, `1024` ou `2048` são mais seguros para testes.

\---

### `--max-num-seqs 1`

Controla quantas sequências/requisições o vLLM pode processar simultaneamente.

No estado atual:

```yaml
--max-num-seqs 1
```

isso significa uma configuração bem conservadora, ideal para validar estabilidade.

Para aumentar acessos simultâneos, teste gradualmente:

```yaml
--max-num-seqs 2
```

Depois:

```yaml
--max-num-seqs 4
```

Mas cada requisição simultânea aumenta o uso de KV cache e pode consumir mais VRAM.

\---

### `--enforce-eager`

Força execução em modo eager do PyTorch, desabilitando algumas otimizações como CUDA Graphs.

Na prática:

* pode reduzir pressão de memória em alguns cenários;
* pode facilitar debugging;
* pode deixar a execução um pouco menos otimizada.

Para GPU pequena, como uma RTX 3050 de 8 GB, é um parâmetro útil para estabilidade inicial.

Depois que o servidor estiver estável, você pode testar remover esse parâmetro e comparar desempenho/memória.

\---

### `shm\_size: "8gb"`

Define o tamanho da memória compartilhada disponível para o container.

Workloads com PyTorch, CUDA e inferência podem precisar de mais memória compartilhada do que o padrão do Docker.

\---

### `deploy.resources.reservations.devices`

Trecho responsável por disponibilizar a GPU NVIDIA para o container:

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: \[gpu]
```

Isso permite que o container enxergue a GPU.

Para verificar dentro do ambiente, use:

```bash
docker exec -it vllm-openai nvidia-smi
```

\---

### `volumes`

```yaml
volumes:
  - E:/vllm-server/models:/models
```

Mapeia uma pasta do Windows para uma pasta dentro do container.

No host:

```text
E:\\vllm-server\\models
```

Dentro do container:

```text
/models
```

Isso é importante para persistir modelos baixados.

\---

### `ports`

```yaml
ports:
  - "8008:8008"
```

Expõe o servidor vLLM do container para o Windows.

Depois de subir, você acessa em:

```text
http://localhost:8008
```

\---

### `environment`

```yaml
environment:
  HF\_TOKEN: ${HF\_TOKEN}
```

Passa o token do Hugging Face para dentro do container.

O valor vem do arquivo `.env`.

\---

## 5\. Sobre o tamanho da imagem `vllm/vllm-openai`

A imagem `vllm/vllm-openai:latest` pode ocupar cerca de **10 GB ou mais** porque ela traz um ambiente completo para inferência com GPU, incluindo bibliotecas de machine learning, CUDA, PyTorch, dependências do vLLM e kernels necessários para execução eficiente.

Esse tamanho é da **imagem Docker**, não do modelo.

Além dela, os modelos baixados ficam na pasta:

```text
E:\\vllm-server\\models
```

Portanto, existem dois consumos de disco diferentes:

```text
Imagem Docker vLLM    → armazenada pelo Docker Desktop
Modelos Hugging Face  → E:\\vllm-server\\models
```

Para ver o tamanho das imagens Docker:

```bash
docker images
```

Para remover imagens antigas/paradas:

```bash
docker system prune
```

Para uma limpeza mais agressiva:

```bash
docker system prune -a
```

Atenção: o comando acima pode remover imagens que você ainda usa em outros projetos.

\---

## 6\. Como iniciar o servidor

Na pasta onde está o `docker-compose.yml`:

```bash
docker compose up -d
```

Ver logs:

```bash
docker logs -f vllm-openai
```

Parar:

```bash
docker compose down
```

\---

## 7\. Testar se o servidor subiu

```bash
curl http://localhost:8008/v1/models
```

Resposta esperada:

```json
{
  "object": "list",
  "data": \[
    {
      "id": "Qwen/Qwen2.5-1.5B-Instruct",
      "object": "model"
    }
  ]
}
```

\---

## 8\. Testar uma chamada de chat via CMD

No CMD do Windows:

```cmd
curl http://localhost:8008/v1/chat/completions ^
  -H "Content-Type: application/json" ^
  -d "{\\"model\\":\\"Qwen/Qwen2.5-1.5B-Instruct\\",\\"messages\\":\[{\\"role\\":\\"user\\",\\"content\\":\\"Explique o que é vLLM em uma frase.\\"}],\\"max\_tokens\\":100}"
```

\---

## 9\. Testar uma chamada de chat via PowerShell

No PowerShell:

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:8008/v1/chat/completions" `
  -Method Post `
  -ContentType "application/json" `
  -Body '{
    "model": "Qwen/Qwen2.5-1.5B-Instruct",
    "messages": \[
      {
        "role": "user",
        "content": "Explique o que é vLLM em uma frase."
      }
    ],
    "max\_tokens": 100
  }'
```

\---

## 10\. Script Python para testar múltiplos acessos simultâneos

Crie um arquivo chamado:

```text
teste\_concorrencia\_vllm.py
```

Instale a biblioteca:

```bash
pip install openai
```

Código:

```python
from concurrent.futures import ThreadPoolExecutor, as\_completed
from time import perf\_counter
from openai import OpenAI

BASE\_URL = "http://localhost:8008/v1"
MODEL = "Qwen/Qwen2.5-1.5B-Instruct"

client = OpenAI(
    base\_url=BASE\_URL,
    api\_key="local-vllm",
)


def chamar\_modelo(i: int) -> dict:
    inicio = perf\_counter()

    try:
        resposta = client.chat.completions.create(
            model=MODEL,
            messages=\[
                {
                    "role": "user",
                    "content": f"Responda em uma frase curta: o que é vLLM? Requisição {i}",
                }
            ],
            max\_tokens=80,
            temperature=0.2,
        )

        fim = perf\_counter()

        return {
            "id": i,
            "ok": True,
            "tempo\_segundos": round(fim - inicio, 2),
            "resposta": resposta.choices\[0].message.content,
        }

    except Exception as erro:
        fim = perf\_counter()

        return {
            "id": i,
            "ok": False,
            "tempo\_segundos": round(fim - inicio, 2),
            "erro": str(erro),
        }


def testar\_concorrencia(total\_requisicoes: int, acessos\_simultaneos: int) -> None:
    print("=" \* 80)
    print(f"Total de requisições: {total\_requisicoes}")
    print(f"Acessos simultâneos: {acessos\_simultaneos}")
    print("=" \* 80)

    inicio\_total = perf\_counter()
    resultados = \[]

    with ThreadPoolExecutor(max\_workers=acessos\_simultaneos) as executor:
        tarefas = \[
            executor.submit(chamar\_modelo, i)
            for i in range(1, total\_requisicoes + 1)
        ]

        for tarefa in as\_completed(tarefas):
            resultado = tarefa.result()
            resultados.append(resultado)

            if resultado\["ok"]:
                print(f"\[OK] Req {resultado\['id']} | {resultado\['tempo\_segundos']}s")
            else:
                print(f"\[ERRO] Req {resultado\['id']} | {resultado\['tempo\_segundos']}s")
                print(resultado\["erro"])

    fim\_total = perf\_counter()
    tempo\_total = round(fim\_total - inicio\_total, 2)

    sucessos = sum(1 for r in resultados if r\["ok"])
    erros = sum(1 for r in resultados if not r\["ok"])

    print("\\n" + "=" \* 80)
    print("RESUMO")
    print("=" \* 80)
    print(f"Sucessos: {sucessos}")
    print(f"Erros: {erros}")
    print(f"Tempo total: {tempo\_total}s")

    if sucessos > 0:
        tempos = \[r\["tempo\_segundos"] for r in resultados if r\["ok"]]
        print(f"Tempo médio por requisição bem-sucedida: {round(sum(tempos) / len(tempos), 2)}s")
        print(f"Menor tempo: {min(tempos)}s")
        print(f"Maior tempo: {max(tempos)}s")

    print("=" \* 80)


if \_\_name\_\_ == "\_\_main\_\_":
    testar\_concorrencia(
        total\_requisicoes=8,
        acessos\_simultaneos=2,
    )
```

Execute:

```bash
python teste\_concorrencia\_vllm.py
```

\---

## 11\. Como interpretar o teste de concorrência

No script:

```python
total\_requisicoes=8
acessos\_simultaneos=2
```

significa:

```text
Enviar 8 requisições no total, com até 2 rodando ao mesmo tempo.
```

Para testar mais carga:

```python
testar\_concorrencia(
    total\_requisicoes=16,
    acessos\_simultaneos=4,
)
```

Mas o compose precisa permitir mais simultaneidade:

```yaml
--max-num-seqs 4
```

Se o compose estiver com:

```yaml
--max-num-seqs 1
```

o vLLM tende a processar de forma mais serializada, mesmo que o Python envie várias chamadas ao mesmo tempo.

\---

## 12\. Sugestões de configuração para testar concorrência

### Configuração estável inicial

```yaml
--gpu-memory-utilization 0.50
--max-model-len 1024
--max-num-seqs 1
--enforce-eager
```

Boa para validar que tudo funciona.

\---

### Teste com 2 acessos simultâneos

```yaml
--gpu-memory-utilization 0.60
--max-model-len 1024
--max-num-seqs 2
--enforce-eager
```

No script Python:

```python
testar\_concorrencia(
    total\_requisicoes=8,
    acessos\_simultaneos=2,
)
```

\---

### Teste com 4 acessos simultâneos

```yaml
--gpu-memory-utilization 0.70
--max-model-len 512
--max-num-seqs 4
--enforce-eager
```

No script Python:

```python
testar\_concorrencia(
    total\_requisicoes=16,
    acessos\_simultaneos=4,
)
```

Se der erro de memória, volte para:

```yaml
--max-num-seqs 2
```

ou reduza:

```yaml
--max-model-len 512
```

\---

## 13\. Ver uso da GPU

Durante o teste, rode:

```bash
nvidia-smi
```

Ou acompanhe em tempo real:

```bash
nvidia-smi -l 1
```

Observe:

* uso de memória da GPU;
* uso de processamento da GPU;
* processos ativos;
* se há outros programas usando VRAM.

\---

## 14\. Problemas comuns

### Erro: `Empty reply from server`

Geralmente significa que o container abriu a porta, mas o servidor vLLM falhou ou ainda está carregando.

Verifique:

```bash
docker logs --tail 120 vllm-openai
```

\---

### Erro: falta de memória para KV cache

Exemplo:

```text
No available memory for the cache blocks
```

Soluções:

* reduzir `--max-model-len`;
* reduzir `--max-num-seqs`;
* fechar programas que usam GPU;
* reduzir ou ajustar `--gpu-memory-utilization`;
* usar modelo menor.

\---

### Erro: `403 Forbidden` no Hugging Face

Significa que o modelo é restrito/gated.

Soluções:

* aceitar os termos do modelo na página do Hugging Face;
* confirmar se o `HF\_TOKEN` está correto;
* usar modelo público, como `Qwen/Qwen2.5-1.5B-Instruct`.

\---

## 15\. Recomendações para RTX 3050 de 8 GB

Para sua GPU, comece com:

```yaml
--gpu-memory-utilization 0.50
--max-model-len 1024
--max-num-seqs 1
--enforce-eager
```

Depois teste:

```yaml
--gpu-memory-utilization 0.60
--max-model-len 1024
--max-num-seqs 2
--enforce-eager
```

Se quiser tentar mais concorrência:

```yaml
--gpu-memory-utilization 0.70
--max-model-len 512
--max-num-seqs 4
--enforce-eager
```

A relação principal é:

```text
mais max-num-seqs = mais concorrência = mais uso de KV cache = mais VRAM
```

E:

```text
mais max-model-len = contexto maior = mais memória por requisição
```

\---

## 16\. Comandos úteis

Subir:

```bash
docker compose up -d
```

Parar:

```bash
docker compose down
```

Ver logs:

```bash
docker logs -f vllm-openai
```

Ver containers:

```bash
docker ps -a
```

Ver imagens:

```bash
docker images
```

Ver uso de GPU:

```bash
nvidia-smi
```

Testar modelos:

```bash
curl http://localhost:8008/v1/models
```

\---

## 17\. Resumo

Este setup executa o modelo `Qwen/Qwen2.5-1.5B-Instruct` localmente com vLLM, usando Docker e GPU NVIDIA.

A API fica disponível em:

```text
http://localhost:8008/v1
```

O servidor é compatível com clientes OpenAI, mas roda localmente.

Para aumentar acessos simultâneos, ajuste principalmente:

```yaml
--max-num-seqs
```

E controle o consumo de memória com:

```yaml
--gpu-memory-utilization
--max-model-len
```

