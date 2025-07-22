# Arquitetura do Pipeline RAG 

Este documento descreve a arquitetura do pipeline de RAG construído para o desafio técnico. O sistema foi projetado para extrair, indexar e consultar informações de documentos heterogêneos (PDFs e imagens) de forma eficiente.

## Visão Geral

O pipeline é composto por três blocos principais, conforme definido no desafio: **1. Extração de Conteúdo**, **2. Indexação** e **3. Recuperação e Geração**. Ele processa documentos de entrada, os transforma em uma base de conhecimento vetorial e utiliza um LLM para responder a perguntas com base no conteúdo recuperado.

## Diagrama da Arquitetura

  [Documentos de Entrada]
  (PDF com texto, Imagem)
           |
           v
+------------------------------------+
| 1. Extração de Conteúdo (Ingestão) |
|------------------------------------|
|   - Se PDF: Extração de texto      |
|     (PyMuPDF)                      |
|   - Se Imagem: Pipeline de OCR     |
|     (Pytesseract + Pós-process.)   |
+------------------------------------+
|
v
[Textos Extraídos]
(Lista de objetos Document)
|
v
+------------------------------------+
|      2. Indexação                  |
|------------------------------------|
| a. Chunking (Divisão de Texto)     |
|    (RecursiveCharacterTextSplitter)|
| b. Embedding (Vetorização)         |
|    (sentence-transformers/         |
|     all-MiniLM-L6-v2)              |
| c. Armazenamento Vetorial (Índice) |
|    (FAISS)                         |
+------------------------------------+
|
v
[Banco Vetorial FAISS]
(Retriever)
|
v
+------------------------------------+
|   3. Recuperação e Geração (RAG)   |
|------------------------------------|
| a. Pergunta do Usuário             |
| b. Recuperação de Chunks Similares |
|    (FAISS Retriever)               |
| c. Augmentation do Prompt          |
|    (LangChain PromptTemplate)      |
| d. Geração de Resposta com LLM     |
|    (Groq API - Llama 3)            |
+------------------------------------+
|
v
[Resposta Final]

## Detalhamento dos Componentes

### 1. Extração de Conteúdo

Esta etapa é responsável por ler os arquivos brutos e extrair seu conteúdo textual[cite: 14].

* **Entrada:** Arquivos heterogêneos, como PDFs com texto nativo e imagens (JPEG, PNG, etc.).
* **Processo:**
    * **Triagem de Arquivos:** O pipeline primeiro identifica o tipo de arquivo pela sua extensão.
    * **PDF com Texto (`.pdf`):** Para PDFs que contêm texto, a biblioteca `PyMuPDF` (`fitz`) é usada para extrair o texto de cada página, que é então convertida em um objeto `Document` do LangChain, mantendo metadados como o nome do arquivo de origem e o número da página.
    * **Imagens (`.webp`, `.png`, etc.):** Para arquivos de imagem, um pipeline de OCR é aplicado:
        1.  **Leitura e Pré-processamento:** A imagem é aberta com a biblioteca `Pillow` e convertida para escala de cinza para melhorar a performance do OCR.
        2.  **Extração com OCR:** O `Pytesseract` processa a imagem e extrai o texto bruto.
        3.  **Pós-processamento Estruturado:** Uma função `structure_ocr_text` foi criada para analisar a saída do OCR (especialmente para tabelas), usando expressões regulares para identificar e formatar cada linha em uma string descritiva e coerente (ex: "Item: [nome] | Código: [código] | ..."). Isso é crucial para preservar o valor semântico dos dados tabulares.
* **Saída:** Uma lista de objetos `Document`, onde cada objeto contém o texto extraído e seus metadados.

### 2. Indexação

Nesta fase, os textos extraídos são transformados em um formato otimizado para busca semântica[cite: 18].

* **Entrada:** A lista de objetos `Document` da etapa anterior.
* **Processo:**
    1.  **Chunking:** Os textos são divididos em fragmentos menores (chunks) usando o `RecursiveCharacterTextSplitter` do LangChain. Essa técnica tenta manter a coesão semântica dividindo o texto em parágrafos ou frases. Foi configurado um tamanho de chunk de 1200 caracteres com uma sobreposição de 300 para garantir que o contexto não seja perdido entre os fragmentos.
    2.  **Geração de Embeddings:** Cada chunk é convertido em um vetor numérico (embedding)  usando o modelo `sentence-transformers/all-MiniLM-L6-v2` da HuggingFace. Este modelo é capaz de capturar o significado semântico do texto, permitindo buscas baseadas em similaridade de conceitos e não apenas em palavras-chave.
    3.  **Armazenamento Vetorial:** Os vetores gerados e seus chunks de texto correspondentes são armazenados em um banco de dados vetorial, o `FAISS` (Facebook AI Similarity Search), uma biblioteca eficiente para busca de similaridade em memória.
* **Saída:** Um índice `FAISS` que funciona como um *retriever*, pronto para buscar chunks relevantes.

### 3. Recuperação e Geração

Esta é a etapa final, onde o sistema deve receber um query para responder perguntas.

* **Entrada:** Uma pergunta do usuário em linguagem natural.
* **Processo (RAG Chain):**
    1.  **Recuperação (Retrieve):** A pergunta do usuário é primeiro convertida em um vetor de embedding (usando o mesmo modelo da etapa de indexação). O `FAISS retriever` então utiliza este vetor para buscar os `k` chunks de texto mais relevantes no banco vetoria. A busca é configurada com o tipo "mmr" (Maximum Marginal Relevance) para garantir diversidade nos resultados recuperados.
    2.  **Aumento (Augment):** Os chunks recuperados são inseridos em um *template* de prompt. Este prompt instrui a LLM a se comportar como um assistente especialista, a basear sua resposta *estritamente* no contexto fornecido e a informar caso não encontre a resposta, citando sempre a fonte.
    3.  **Geração (Generate):** O prompt completo (contexto + pergunta) é enviado para a LLM. A solução utiliza o modelo `llama3-70b-8192` através da API da Groq. A LLM, então, gera uma resposta em linguagem natural com base nas informações recuperadas.
* **Saída:** A resposta final para a pergunta do usuário.
