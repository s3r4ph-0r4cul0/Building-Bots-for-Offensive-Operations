# Documentação: Agente de IA para Reconhecimento Ofensivo com Nmap

## 1. Introdução

Este documento detalha a arquitetura, implementação e uso de um agente de Inteligência Artificial projetado para realizar operações de reconhecimento de rede utilizando a ferramenta Nmap. O agente incorpora um Large Language Model (LLM) para otimizar comandos Nmap e um sistema de auto-treinamento para refinar suas estratégias com base nos resultados das varreduras.

## 2. Arquitetura do Sistema

### 2.1. Componentes Principais

### 2.1.1. Orquestrador/Interface do Usuário
- **Função**: Ponto de entrada para o usuário, gerenciamento do fluxo de trabalho, iniciação de varreduras e exibição de resultados.
- **Tecnologia Sugerida**: Python com Flask/FastAPI para uma API RESTful ou uma interface de linha de comando (CLI) interativa.

### 2.1.2. Agente de IA (LLM-powered)
- **Função**: Traduzir objetivos de reconhecimento de alto nível em comandos Nmap otimizados, interpretar resultados do Nmap e gerar insights.
- **Tecnologia Sugerida**: Um Large Language Model (LLM) como GPT-4 (via API) ou um modelo open-source como Llama 3 (se houver recursos para hospedagem local).

### 2.1.3. Executor Nmap
- **Função**: Executar comandos Nmap no ambiente containerizado e capturar a saída padrão (stdout) e erro padrão (stderr).
- **Tecnologia Sugerida**: Subprocessos Python para chamar o Nmap diretamente.

### 2.1.4. Analisador de Saída (Output Analyzer)
- **Função**: Processar e analisar a saída bruta do Nmap (XML ou formato normal), extraindo informações estruturadas como portas abertas, serviços, versões, detecção de SO, etc.
- **Tecnologia Sugerida**: Bibliotecas Python como `python-nmap` (para parsing de XML) ou regex para parsing de saída normal.

### 2.1.5. Módulo de Treinamento e Refinamento (Self-Training & Refinement Module)
- **Função**: Avaliar a eficácia dos comandos Nmap executados, refinar estratégias de varredura com base nos resultados e atualizar a base de conhecimento do agente.
- **Tecnologia Sugerida**: Algoritmos de Reinforcement Learning (RL) ou técnicas de aprendizado por feedback para ajustar os prompts do LLM ou os parâmetros do Nmap. Um banco de dados para armazenar resultados de varreduras e métricas de sucesso.

### 2.1.6. Base de Conhecimento/Banco de Dados
- **Função**: Armazenar dados de varreduras anteriores, perfis de alvos, resultados de análises, e aprimoramentos de estratégias de Nmap para o auto-treinamento.
- **Tecnologia Sugerida**: PostgreSQL ou MongoDB para flexibilidade e escalabilidade.

## 2.2. Fluxo de Dados e Interações

1.  **Usuário/Orquestrador**: O usuário define um objetivo de reconhecimento (ex: 'escanear portas abertas em 192.168.1.1').
2.  **Orquestrador -> Agente de IA**: O Orquestrador envia o objetivo para o Agente de IA.
3.  **Agente de IA -> Executor Nmap**: O Agente de IA, utilizando o LLM e sua base de conhecimento, gera um comando Nmap otimizado e o envia para o Executor Nmap.
4.  **Executor Nmap -> Nmap**: O Executor Nmap executa o comando Nmap.
5.  **Nmap -> Executor Nmap**: O Nmap retorna o output (XML ou texto) para o Executor Nmap.
6.  **Executor Nmap -> Analisador de Saída**: O Executor Nmap encaminha o output bruto para o Analisador de Saída.
7.  **Analisador de Saída -> Módulo de Treinamento e Refinamento**: O Analisador de Saída processa o output e envia os dados estruturados para o Módulo de Treinamento e Refinamento.
8.  **Módulo de Treinamento e Refinamento -> Base de Conhecimento**: O Módulo de Treinamento e Refinamento avalia os resultados, refina a estratégia e armazena os dados e insights na Base de Conhecimento.
9.  **Módulo de Treinamento e Refinamento -> Agente de IA**: O Módulo de Treinamento e Refinamento pode fornecer feedback ou atualizações para o Agente de IA (ex: ajustar prompts do LLM para futuras varreduras).
10. **Analisador de Saída -> Orquestrador**: O Analisador de Saída envia os resultados estruturados para o Orquestrador para exibição ao usuário.

## 3. Containerização com Docker

O ambiente do agente é containerizado usando Docker para garantir portabilidade, isolamento e fácil implantação. O `Dockerfile` define o ambiente de execução e as dependências necessárias.

### 3.1. Dockerfile

```dockerfile
FROM python:3.9-slim-buster

# Instalação do Nmap
RUN apt-get update && apt-get install -y nmap && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "main.py"]
```

**Explicação:**
- `FROM python:3.9-slim-buster`: Define a imagem base como Python 3.9 em uma distribuição Debian Buster slim, que é leve.
- `RUN apt-get update && apt-get install -y nmap && rm -rf /var/lib/apt/lists/*`: Atualiza os pacotes e instala o Nmap, removendo arquivos temporários para manter a imagem pequena.
- `WORKDIR /app`: Define o diretório de trabalho dentro do container.
- `COPY requirements.txt .`: Copia o arquivo `requirements.txt` para o diretório de trabalho.
- `RUN pip install --no-cache-dir -r requirements.txt`: Instala as dependências Python listadas no `requirements.txt`.
- `COPY . .`: Copia todo o conteúdo do diretório local para o diretório de trabalho do container.
- `CMD ["python", "main.py"]`: Define o comando a ser executado quando o container é iniciado.

### 3.2. `requirements.txt`

```
python-nmap
openai
flask
```

## 4. Implementação do Agente de IA (main.py)

O arquivo `main.py` contém a lógica principal do agente, incluindo a interação com o LLM, execução do Nmap, análise de saída e o módulo de auto-treinamento.

### 4.1. Geração de Comandos Nmap (`generate_nmap_command`)

Esta função utiliza um LLM (como GPT-4) para traduzir um objetivo de reconhecimento em um comando Nmap otimizado. O prompt instrui o LLM a ser preciso e a incluir a opção de saída em XML (`-oX`).

### 4.2. Execução do Nmap (`execute_nmap_scan`)

Esta função executa o comando Nmap gerado pelo LLM. Ela utiliza a biblioteca `python-nmap` para interagir com o Nmap, facilitando a execução e a captura da saída. A função tenta extrair o alvo e os argumentos do comando gerado pelo LLM e executa a varredura, retornando a saída XML e um resumo dos hosts varridos.

### 4.3. Análise de Saída do Nmap (`analyze_nmap_output`)

Após a execução do Nmap, esta função processa a saída XML para extrair informações estruturadas. Ela utiliza `python-nmap` para analisar o XML e retornar detalhes como IPs, hostnames, estados dos hosts, protocolos, portas abertas, estados das portas, nomes de serviços, produtos e versões.

### 4.4. Refinamento de Comandos Nmap e Base de Conhecimento (`refine_nmap_command`, `load_knowledge_base`, `save_knowledge_base`)

O módulo de auto-treinamento é implementado através da função `refine_nmap_command` e de uma base de conhecimento persistente (`knowledge_base.json`).

- `refine_nmap_command`: Esta função recebe o objetivo original, o comando Nmap anterior e a análise do output. Ela utiliza o LLM para sugerir um comando Nmap mais refinado, com base na eficácia da varredura anterior. O LLM é instruído a focar em obter mais detalhes ou cobrir lacunas identificadas.
- `load_knowledge_base` e `save_knowledge_base`: Funções utilitárias para carregar e salvar a base de conhecimento em um arquivo JSON. A base de conhecimento armazena entradas contendo o objetivo, o comando anterior, a análise do output e o comando refinado, permitindo que o agente aprenda e melhore suas estratégias ao longo do tempo.

### 4.5. Endpoints da API (Flask)

O `main.py` expõe dois endpoints principais via Flask:

- `/scan` (POST): Recebe um objetivo de reconhecimento, gera e executa um comando Nmap, e retorna o comando gerado, a saída bruta e a saída analisada do Nmap.
- `/refine_scan` (POST): Recebe o objetivo, o comando anterior e a análise do output, refina o comando Nmap usando o LLM e armazena a entrada na base de conhecimento. Retorna o comando Nmap refinado e a entrada da base de conhecimento.

## 5. Como Usar

### 5.1. Pré-requisitos
- Docker instalado.
- Chave de API da OpenAI (ou acesso a um LLM local).

### 5.2. Construção da Imagem Docker

Navegue até o diretório `offensive_recon_agent` e construa a imagem Docker:

```bash
docker build -t offensive-recon-agent .
```

### 5.3. Execução do Container

Execute o container, passando sua chave de API da OpenAI como variável de ambiente:

```bash
docker run -p 5000:5000 -e OPENAI_API_KEY="sua_chave_api_openai" offensive-recon-agent
```

O agente estará acessível na porta 5000.

### 5.4. Exemplo de Uso da API

#### 5.4.1. Iniciar uma Varredura

Envie uma requisição POST para `/scan`:

```bash
curl -X POST -H "Content-Type: application/json" -d '{"objective": "escanear portas abertas em 192.168.1.1"}' http://localhost:5000/scan
```

#### 5.4.2. Refinar uma Varredura

Após obter os resultados de uma varredura, você pode usar o endpoint `/refine_scan` para refinar o comando. Você precisará do objetivo original, do comando Nmap gerado e da saída analisada da varredura anterior.

```bash
curl -X POST -H "Content-Type: application/json" -d '{
    "objective": "escanear portas abertas em 192.168.1.1",
    "previous_command": "nmap -sS 192.168.1.1 -oX -",
    "analyzed_output": { "hosts": [ { "ip": "192.168.1.1", "state": "up", "protocols": [] } ] }
}' http://localhost:5000/refine_scan
```

## 6. Considerações de Segurança

É fundamental operar este agente em um ambiente controlado e isolado (ex: rede de laboratório, máquinas virtuais dedicadas) para evitar o uso indevido ou impactos não intencionais em sistemas reais. O uso de containers Docker ajuda no isolamento, mas não substitui a necessidade de um ambiente de teste seguro e ético. Certifique-se de ter permissão legal e ética para realizar qualquer varredura de rede.

## 7. Próximos Passos

- Implementar uma interface de usuário mais amigável (web ou CLI).
- Expandir a base de conhecimento com mais exemplos de varreduras e refinamentos.
- Integrar com outras ferramentas de segurança para um fluxo de trabalho mais completo.
- Adicionar validação de entrada mais robusta para os comandos Nmap gerados pelo LLM.



