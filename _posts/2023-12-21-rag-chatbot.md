---
title: Retrieval-Augmented Generation (RAG) ChatBot
date: 2023-12-21 3:00:00 -0500
categories: [Development]
tags: [flask, react, api, vector database, llm]     # TAG names should always be lowercase
---

## Result

![](/assets/media/chatbot_result.gif)

## Backend

- designed by Flask
    - contains two URLs: "/setup" and "/chat"
    - "/setup" is used for init connection with HuggingFace/OpenAI and Pinecone
    - "/chat" is used for processing message from client

- chat service
    - Using LLM model 'google/flan-t5-xxl' and embedding model 'intfloat/e5-small-v2' from HuggingFace
    ```python
    {
        "embedding": HuggingFaceInstructEmbeddings(model_name="intfloat/e5-small-v2"),
        "dimension": 384,
        "metric": "cosine",
        "chat": HuggingFaceHub(repo_id="google/flan-t5-xxl", model_kwargs={"temperature":0.7, "max_length":512})
    }
    ```
    - Option: LLM model ChatGPT and embedding model 'text-embedding-ada-002' from OpenAI (will get better result)
    ```python
    {
        "embedding": OpenAIEmbeddings(model="text-embedding-ada-002"),
        "dimension": 1536,
        "metric": "cosine",
        "chat": ChatOpenAI(
                    openai_api_key=os.environ["OPENAI_API_KEY"],
                    model='gpt-3.5-turbo'
                )
    }
    ```
    - Use vectordatabase Pinecone to ensure persistence, avoid extract documents every time
    ```py
    import pinecone
    pinecone.init(
        api_key=os.getenv('PINECONE_API_KEY'),
        environment=os.getenv('PINECONE_ENVIRONMENT')
    )
    ```
    - Use Spacy to extract keywords from user question to get a better query result from Pinecone
    ```py
    import spacy
    # extract keywords from question, en-core-web-trf is focused on accuracy
    nlp = spacy.load("en_core_web_trf")
    doc = nlp(query)
    # Extract keywords based on POS tags (e.g., NOUN, PROPN for proper nouns, VERB)
    keywords = [token.text for token in doc if token.pos_ in ('NOUN', 'PROPN', 'VERB')]
    ```
    - Customize prompt to allow LLM refer the knowledges from vector database and the chat history

    ```py
    source_knowledge = "\n".join([x.page_content for x in results])
    history = "\n".join([x.content for x in chat_history])
    augmented_prompt = f"""Based on the contexts below and the chat history, conclude the contexts and answer the question in your own words. You can answer you don't know if you cannot find related information from contexts.

    Contexts:
    {source_knowledge}.

    Chat history:
    {history}.

    Question: {query}"""
    ```

## Frontend

- designed by React and refered to GPT's UI
- setup the chat history like the backend
- using axios to make request to the backend

## Source Code

Available on [Github](https://github.com/zhaojun-szh-9815/rag-chatbot)

```bash
git clone git@github.com:zhaojun-szh-9815/rag-chatbot.git
cd rag-chatbot
```
```bash
cd server
pip install -r requirements.txt
flask run
```
```bash
cd client
npm install
npm start
```