# 🪢 **NFe & Audio Automation Workflows (n8n)**
Este repositório contém a lógica de orquestração e o "sistema nervoso" do ecossistema Menegolo Data Stack. Utilizando o n8n, os fluxos aqui presentes automatizam a captura de despesas via Telegram, processando desde imagens de notas fiscais (OCR/Scraping) até notas de voz (IA/NLP).

## 🛰️ **Dependência: NFe Scraper API (Engine de Visão)**
O fluxo de processamento de imagem (02-image-processor) depende obrigatoriamente da **NFe Scraper API.**
- **Deploy Automatizado:** Esta API possui um pipeline de CI/CD que realiza o build automático de imagens Docker.
- **Imagem Oficial:** Disponível no GitHub Container Registry:
    - image: ghcr.io/fmenegolo/nfe-scraper-api:lite
- **Função no Fluxo:** Recebe o binário da imagem do n8n, utiliza OpenCV (WeChatQR) para decodificação e retorna o JSON estruturado após o scraping via Playwright.
- Você pode conferir o código-fonte ou clonar no GitHub: https://github.com/fmenegolo/nfe-scraper-api.
- **Atenção: Sem esta API rodando na porta 8000/8001, o fluxo 02-image-processor retornará erro de conexão.**

## 🏗️ **Arquitetura Modular (Router & Workers)**
Para garantir alta disponibilidade e facilitar a manutenção, a automação foi dividida em três fluxos independentes que se comunicam via nós de Execute Workflow.

### 🚦 01-router.json
**O Gateway de Entrada.**
- Gatilho: Webhook do Telegram Bot.
- Inteligência: Utiliza um nó Switch para analisar o MIME-Type da mensagem.
- Roteamento:
    - audio/ogg ➡️ Encaminha para o Processador de Áudio.
    - photo (Array) ➡️ Encaminha para o Processador de Imagem (NFe).
- Resiliência: Possui um fluxo de erro global para notificações de falha.
### 🖼️ 02-image-processor.json
**O Worker de Notas Fiscais.**
- Integração: Consome a nfe-scraper-api (FastAPI/Playwright) via requisições HTTP.
- Lógica de Arquivo: Transforma o JSON retornado pela API em arquivos Markdown estruturados.
- Padronização: Nomeia arquivos no formato AAAAMMDD_HHMM_Categoria_Valor_S.md.
- Relacionamento: Utiliza Split Out para criar notas individuais para cada item da compra, linkando-os ao arquivo "Pai" da nota fiscal.
### 🎙️ 03-audio-processor.json
**O Worker de Inteligência de Voz.**
- IA Generativa: Utiliza Google Gemini 2.5 Flash para transcrição e extração de entidades financeiras.
- Processamento: Um motor em JavaScript limpa o texto (remoção de acentos/símbolos) e calcula o valor_saldo (positivo/negativo).
- Contexto: A IA interpreta datas relativas ("ontem", "hoje") cruzando com o timestamp da mensagem original.

## 🛠️ **Tecnologias Utilizadas**
- Orquestrador: n8n (Self-hosted no CasaOS).
- IA: Google Gemini SDK (LangChain nodes).
- Mensageria: Telegram Bot API.
- Storage: Obsidian Vault (via Docker Volumes).

## 🚀 **Como Importar para o seu n8n**
A simples importação dos arquivos não é suficiente; os fluxos precisam interagir. Siga esta ordem:
1. Importe os Workers primeiro:
    - Crie dois novos workflows e importe os arquivos 02-image-processor.json e 03-audio-processor.json.
    - Salve-os e anote o ID que o n8n gerou para cada um (está na URL do navegador, ex: workflow/12).
2. Importe o Router:
    - Crie um terceiro workflow e importe o 01-router.json.
3. Configure as Conexões Internas:
    - No fluxo 01-router, abra os nós de Execute Workflow.
    - No campo Workflow ID, selecione os fluxos salvos no passo 1. Isso garante que o roteador saiba para quem enviar a imagem ou o áudio.
4. Ajuste as Credenciais:
    - Configure suas chaves de API para Telegram e Google Gemini em seus respectivos nós.
5. Ative os Workflows:
    - Clique no botão Active em todos os 3 fluxos para que o Webhook do Telegram comece a escutar.
------
### 🤝 **Créditos e Inspiração**
Este projeto utiliza componentes e conceitos de automação inspirados em comunidades open-source:
- O fluxo de processamento de áudio foi inicialmente adaptado a partir do trabalho de **Kizzy Terra**, sendo posteriormente refatorado para integração nativa com Obsidian e lógica de persistência em Modern Data Stack.
