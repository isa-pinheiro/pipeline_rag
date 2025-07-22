# Desafio Técnico: Pipeline RAG

Este repositório contém a solução para o desafio técnico, que consiste na construção de um pipeline RAG.

## Objetivo do Projeto

O objetivo é desenvolver um pipeline funcional capaz de:
* Indexar documentos de formatos variados (PDFs com texto e imagens).
* Aplicar Reconhecimento Óptico de Caracteres (OCR) em imagens.
* Criar uma base de dados vetorial para busca semântica.
* Implementar um sistema de recuperação de informação que, a partir de uma pergunta, busca os trechos mais relevantes nos documentos.
* Utilizar uma LLM para gerar respostas baseadas exclusivamente nos documentos recuperados.

## Arquitetura e Escolhas Técnicas

A solução foi implementada como um Jupyter Notebook para facilitar a prototipagem e a visualização dos resultados de cada etapa. A arquitetura segue o padrão RAG (Retrieval-Augmented Generation).

### 1. Extração de Conteúdo

* **PDF com Texto (`CÓDIGO DE OBRAS.pdf`):** Foi utilizada a biblioteca **PyMuPDF (`fitz`)** pela sua boa performance e precisão na extração de texto nativo de arquivos PDF, preservando a estrutura básica e metadados como o número da página, essa biblioteca funciona extraindo pagina por página.
* **Imagem (`tabela.webp`):** Para a imagem contendo uma tabela, foi criado um pipeline de OCR:
    * **OCR:** **Pytesseract** foi escolhido por ter bom suporte para a língua portuguesa (`por`) e ser eficaz na extração de texto em imagens.
    * **Pós-processamento:** A saída bruta do Tesseract para tabelas pode ser desestruturada. Para contornar isso, foi implementada uma função com `regex` para analisar o texto extraído, identificar as colunas (código, artigo, preço, etc.) e reconstruir cada linha da tabela em uma frase semanticamente rica. Essa abordagem melhora drasticamente a qualidade do contexto fornecido ao LLM.

### 2. Indexação

* **Chunking:** Utilizando o **`RecursiveCharacterTextSplitter`** da LangChain. Ele é robusto por tentar dividir o texto em separadores lógicos (parágrafos, frases) antes de recorrer a uma divisão por caracteres. A sobreposição (`chunk_overlap=300`) é uma escolha para garantir que o contexto semântico não seja perdido nas bordas dos chunks.
* **Embeddings:** O modelo escolhido foi o **`sentence-transformers/all-MiniLM-L6-v2`** através da biblioteca `HuggingFaceEmbeddings`. 
* **Banco Vetorial:** **FAISS (Facebook AI Similarity Search)** foi a escolha para o armazenamento dos vetores. Por ser executado localmente, elimina a necessidade de configurar um serviço de banco de dados externo.

### 3. Recuperação e Geração

* **Retriever:** O índice FAISS foi configurado como um retriever no LangChain usando a busca **"Maximum Marginal Relevance" (MMR)**. Diferente da busca por similaridade padrão, que pode retornar chunks muito parecidos entre si, o MMR busca um equilíbrio entre a relevância para a pergunta e a diversidade dos documentos recuperados, fornecendo um contexto mais rico e variado para o LLM.
* **LLM (Grande Modelo de Linguagem):** A escolha foi pelo modelo **`llama3-70b-8192`** acessado via **API da Groq**. A Groq é focada em fornecer inferência de LLMs com latência muito baixa, o que torna a experiência de consulta mais fluida e interativa, além de não estar rodando localmente então em termos de recursos há economia, como revés tem o fato de não ter certeza da privacidade dos dados.
* **Prompt Engineering:** Um template de prompt detalhado foi criado para guiar o comportamento do LLM. As instruções são explícitas:
    1.  Agir como um assistente especialista.
    2.  Basear a resposta *apenas* no contexto fornecido (`{context}`).
    3.  Se a resposta não estiver no contexto, declarar isso explicitamente em vez de "alucinar" ou usar conhecimento externo.
    4.  Citar a fonte (arquivo e página) da informação.
* **Chain (Cadeia):** A lógica RAG foi montada usando o **LangChain**, combinando os componentes (`retriever`, `prompt`, `llm`, `parser`) de forma declarativa e modular, facilitando a manutenção e a compreensão do fluxo de dados.

## Como Executar

O projeto foi desenvolvido para ser executado no ambiente do Google Colab, mas com algumas adaptações pode ser executado localmente, em dois pontos principais no notebook tem o que deve ser feito para que seja feito isso.

### Pré-requisitos

1.  Uma conta Google com acesso ao Google Drive e ao Google Colab.
2.  Chaves de API para:
    * **Groq:** Para acessar o LLM Llama 3.
    * **Hugging Face:** Para baixar o modelo de embeddings.

#### Localmente
Caso o notebook seja executado localmente, algumas alterações devem ser feitas:
1. Algumas linhas de código devem ser comentadas e outras devem ser descomentadas (presentes no notebook ipynb) 
- as referentes aos caminhos do arquivo pdf e de imagem
- as referentes a definir as chaves de API do Groq e do Hugging Face
2. Um arquivo .env deve ser criado contendo as chaves
3. Deve ser executado um `pip install -r requirements.txt`para que as dependências necessárias sejam instaladas
4. O tesseract deverá ser instalado, para o linux a linha de código está em uma célula no notebook

### Passos para Execução

1.  **Upload dos Arquivos:**
    * Faça o upload do notebook (`Desafio_Técnico_Nuven.ipynb`) para o seu Google Colab.
    * Crie uma pasta chamada `nuven` no MyDrive do Google Drive.
    * Faça o upload dos documentos fornecidos (`CÓDIGO DE OBRAS.pdf` e `tabela.webp`) para a pasta `nuven`.

2.  **Configuração do Ambiente no Colab:**
    * Abra o notebook no Google Colab.
    * No painel esquerdo, clique no ícone de chave (Secrets) e adicione suas chaves de API:
        * `GROQ_API_KEY`: Sua chave da API da Groq.
        * `HF_TOKEN`: Seu token de acesso da Hugging Face.
    * **Atenção:** O código está configurado para ler as chaves a partir do gerenciador de secrets do Colab.

3.  **Execução do Notebook:**
    * Execute as células do notebook em ordem sequencial.
    * **Montagem do Drive:** Você precisará autorizar o acesso do Colab ao seu Google Drive.
    * **Células de Instalação:** As dependências necessárias, incluindo `pytesseract` e as bibliotecas Python, serão instaladas no ambiente.
    * **Células de Execução:** As etapas de extração, indexação e configuração da cadeia RAG serão executadas.
    * **Seção de Demonstração:** Ao final, você pode modificar ou adicionar novas perguntas na seção "Demonstração" para interagir com o pipeline e testar suas capacidades.