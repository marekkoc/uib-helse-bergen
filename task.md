# Solution for a Knowledge-Based Chatbot System for FIONA Application

Based on the provided documents, I'll design a chatbot solution for the FIONA application that will help users find information about planning research projects and features of the Steve system.

## 1. Approach to Solution and Preventing Hallucinations

### System Architecture
I propose an architecture based on RAG (Retrieval-Augmented Generation) models:

1. **Document Indexing**:
   - Processing the provided documents into text fragments (chunking)
   - Creating a vector knowledge base from FIONA/Steve documents
   - Generating embeddings using a multilingual vectorization model

2. **Context Retrieval**:
   - User query is vectorized
   - System searches for the most relevant fragments from the knowledge base
   - Application of semantic filtering to increase relevance

3. **Response Generation**:
   - Context + user query passed to LLM
   - Model generates response based on the provided context

### Preventing Hallucinations

1. **Source Verification**:
   - Each response includes reference to specific document fragments
   - System shows which parts of the documents were used to generate the response

2. **Limiting Model Creativity**:
   - Instructions for the model prohibiting speculation on topics absent from the documents
   - System admits lack of knowledge when the query goes beyond the scope of documents

3. **Citation Mechanism**:
   - Direct quotes from source documents in responses
   - References to specific sections of documentation

## 2. CI/CD Pipeline for Document Updates

### Automatic Knowledge Base Updates

1. **Monitoring Changes in Documents**:
   - Scripts observing document repository
   - Change detection by comparing checksums or metadata

2. **Update Pipeline**:
   ```
   Change detection → Processing new/modified documents → 
   Updating vector database → Regression tests → Deployment
   ```

3. **Knowledge Versioning**:
   - Each version of the knowledge base receives a unique identifier
   - Ability to restore previous versions in case of problems

4. **Automated Testing**:
   - Set of test queries for verifying response correctness
   - Comparison of responses before and after update to detect regression

## 3. Deployment on Different Hardware Platforms

### Containerization

1. **Docker-Based Architecture**:
   - Separate containers for:
     - Frontend server (UI)
     - Backend (API)
     - Vector database
     - LLM model (locally or proxy to API)

2. **Hardware Requirements**:
   - Minimal configuration: 4 CPU cores, 8GB RAM
   - Recommended configuration: 8+ CPU cores, 16+ GB RAM, GPU (for local LLM)

3. **Scaling**:
   - Adaptation to available resources through configuration
   - Ability to run in "proxy" mode (using external model APIs)
   - Option to reduce model size for weaker machines

### Deployment Automation

1. **Installation Scripts**:
   - Bash scripts automating installation on Debian/Ubuntu
   - Ansible playbooks for managing multiple instances

2. **Monitoring**:
   - Performance metrics and resource utilization
   - Alerts in case of operation problems

## 4. User Interface

### UI Features

1. **Responsive Design**:
   - Adaptation to different screen sizes
   - Support for both computers and mobile devices

2. **Interactive Chat**:
   - Real-time text input
   - Typing indicator during response generation
   - Conversation history with scrolling capability

3. **Multilingual Support**:
   - Automatic detection of query language
   - Interface available in various European languages
   - Language switch for preferred response language

4. **UI Elements Enhancing Trust**:
   - Display of response sources
   - Ability to expand detailed quotes
   - Response usefulness rating by user

### Frontend Technologies

1. **Framework**: React or Vue.js with TypeScript
2. **Styling**: Tailwind CSS for modern look
3. **Communication**: WebSockets for real-time responses

## 5. Integration with Existing System (Apache2/PHP 8.x)

### Integration Methods

1. **Reverse Proxy**:
   - Apache2 configuration as a proxy for chatbot application
   - Maintaining consistent URL and user session

2. **Embedding in Existing Application**:
   - JavaScript widget for embedding on existing pages
   - API communication between PHP and chatbot backend

3. **Authentication**:
   - Using existing authentication mechanisms (PHP sessions)
   - Passing authentication token to chatbot API

### Implementation Details

1. **Apache2 Configuration**:
   ```apache
   <VirtualHost *:80>
       # Existing PHP configuration
       # ...
       
       # Proxy for chatbot
       ProxyPass /chatbot http://localhost:3000
       ProxyPassReverse /chatbot http://localhost:3000
       
       # CORS for API
       <Location /api/chatbot>
           Header set Access-Control-Allow-Origin "*"
           Header set Access-Control-Allow-Methods "GET, POST, OPTIONS"
           Header set Access-Control-Allow-Headers "Content-Type, Authorization"
       </Location>
   </VirtualHost>
   ```

2. **PHP Integration**:
   ```php
   <?php
   // Function for communicating with chatbot API
   function queryBot($text, $language = 'en') {
       $ch = curl_init('http://localhost:3000/api/query');
       $data = json_encode([
           'query' => $text,
           'language' => $language,
           'user_id' => $_SESSION['user_id'] ?? 'anonymous'
       ]);
       
       curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
       curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
       curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
       
       $response = curl_exec($ch);
       curl_close($ch);
       
       return json_decode($response, true);
   }
   ?>
   ```

## Implementation Codes

### 1. Document Indexing and Knowledge Base Building

```python
# indexer.py - script for building knowledge base
import os
from langchain.document_loaders import PyPDFLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.vectorstores import FAISS

# Configuration of multilingual embedding model
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/paraphrase-multilingual-mpnet-base-v2",
    model_kwargs={'device': 'cuda'} if torch.cuda.is_available() else {'device': 'cpu'}
)

def load_documents(docs_dir):
    docs = []
    for file in os.listdir(docs_dir):
        path = os.path.join(docs_dir, file)
        if file.endswith('.pdf'):
            loader = PyPDFLoader(path)
            docs.extend(loader.load())
        elif file.endswith('.txt') or file.endswith('.md'):
            loader = TextLoader(path)
            docs.extend(loader.load())
    return docs

def process_documents(documents):
    # Splitting documents into smaller chunks
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=50,
        separators=["\n\n", "\n", ".", " ", ""]
    )
    chunks = text_splitter.split_documents(documents)
    
    # Creating vector database
    vectorstore = FAISS.from_documents(chunks, embeddings)
    vectorstore.save_local("knowledge_base")
    
    print(f"Indexed {len(chunks)} document fragments")

if __name__ == "__main__":
    documents = load_documents("./docs")
    process_documents(documents)
```

### 2. Chatbot Backend (Flask API)

```python
# app.py - chatbot backend
from flask import Flask, request, jsonify
from flask_cors import CORS
import torch
from langchain.vectorstores import FAISS
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.llms import HuggingFaceHub
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate

app = Flask(__name__)
CORS(app)

# Loading embedding model
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/paraphrase-multilingual-mpnet-base-v2",
    model_kwargs={'device': 'cuda'} if torch.cuda.is_available() else {'device': 'cpu'}
)

# Loading knowledge base
db = FAISS.load_local("knowledge_base", embeddings)
retriever = db.as_retriever(search_kwargs={"k": 3})

# LLM model configuration
llm = HuggingFaceHub(
    repo_id="mistralai/Mistral-7B-Instruct-v0.2",
    model_kwargs={"temperature": 0.2, "max_length": 1024}
)

# Prompt template preventing hallucinations
template = """
You are an assistant for the FIONA system and Steve project. Answer the user's question solely based on the provided context.
If the answer is not found in the context, say: "I'm sorry, I can't find information about this in the FIONA/Steve documentation."
Do not make up information that is not contained in the context.
Context:
{context}

Question: {question}
Answer:
"""
QA_PROMPT = PromptTemplate(template=template, input_variables=["context", "question"])

# Creating QA chain
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=retriever,
    chain_type_kwargs={"prompt": QA_PROMPT}
)

@app.route('/api/query', methods=['POST'])
def query_bot():
    data = request.json
    query = data.get('query', '')
    language = data.get('language', 'en')
    
    # Retrieving source documents for transparency
    docs = retriever.get_relevant_documents(query)
    sources = [{"text": doc.page_content, "source": doc.metadata.get('source', 'unknown')} for doc in docs]
    
    # Generating response
    result = qa_chain({"query": query})
    
    return jsonify({
        "answer": result["result"],
        "sources": sources
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3000)
```

### 3. React Frontend with TypeScript

```typescript
// ChatComponent.tsx
import React, { useState, useEffect, useRef } from 'react';
import './ChatComponent.css';

interface Message {
  id: string;
  text: string;
  isUser: boolean;
  sources?: {
    text: string;
    source: string;
  }[];
}

const ChatComponent: React.FC = () => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [language, setLanguage] = useState('en');
  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    // Initial welcome message
    setMessages([
      {
        id: '1',
        text: 'Hello! I am a FIONA assistant. How can I help with your research project?',
        isUser: false
      }
    ]);
  }, []);

  useEffect(() => {
    // Auto-scroll to latest message
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!input.trim()) return;

    // Adding user message
    const userMessage: Message = {
      id: Date.now().toString(),
      text: input,
      isUser: true
    };
    setMessages(msgs => [...msgs, userMessage]);
    setInput('');
    setIsLoading(true);

    try {
      const response = await fetch('http://localhost:3000/api/query', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          query: input,
          language: language
        })
      });

      const data = await response.json();
      
      // Adding bot response
      const botMessage: Message = {
        id: (Date.now() + 1).toString(),
        text: data.answer,
        isUser: false,
        sources: data.sources
      };
      
      setMessages(msgs => [...msgs, botMessage]);
    } catch (error) {
      console.error('Error fetching response:', error);
      setMessages(msgs => [...msgs, {
        id: (Date.now() + 1).toString(),
        text: 'Sorry, an error occurred while getting a response.',
        isUser: false
      }]);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="chat-container">
      <div className="language-selector">
        <select value={language} onChange={(e) => setLanguage(e.target.value)}>
          <option value="en">English</option>
          <option value="pl">Polski</option>
          <option value="de">Deutsch</option>
          <option value="fr">Français</option>
        </select>
      </div>
      
      <div className="messages-container">
        {messages.map(message => (
          <div 
            key={message.id} 
            className={`message ${message.isUser ? 'user-message' : 'bot-message'}`}
          >
            <div className="message-text">{message.text}</div>
            
            {message.sources && (
              <div className="sources-container">
                <button className="sources-toggle">Show sources</button>
                <div className="sources-content">
                  {message.sources.map((source, idx) => (
                    <div key={idx} className="source-item">
                      <div className="source-text">{source.text}</div>
                      <div className="source-file">Source: {source.source}</div>
                    </div>
                  ))}
                </div>
              </div>
            )}
          </div>
        ))}
        
        {isLoading && (
          <div className="typing-indicator">
            <span></span>
            <span></span>
            <span></span>
          </div>
        )}
        
        <div ref={messagesEndRef} />
      </div>
      
      <form onSubmit={handleSubmit} className="input-form">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Type a message..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading || !input.trim()}>
          Send
        </button>
      </form>
    </div>
  );
};

export default ChatComponent;
```

### 4. CSS for Chat Component

```css
/* ChatComponent.css */
.chat-container {
  display: flex;
  flex-direction: column;
  height: 600px;
  width: 400px;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  background-color: #f9f9f9;
  overflow: hidden;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.language-selector {
  padding: 10px;
  background-color: #007bff;
  color: white;
}

.language-selector select {
  padding: 5px;
  border-radius: 5px;
  border: none;
  background-color: white;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.message {
  max-width: 80%;
  padding: 10px 15px;
  border-radius: 18px;
  word-wrap: break-word;
}

.user-message {
  align-self: flex-end;
  background-color: #007bff;
  color: white;
  border-bottom-right-radius: 4px;
}

.bot-message {
  align-self: flex-start;
  background-color: #e6e6e6;
  color: #333;
  border-bottom-left-radius: 4px;
}

.typing-indicator {
  align-self: flex-start;
  background-color: #e6e6e6;
  border-radius: 18px;
  padding: 10px 15px;
  display: flex;
  gap: 4px;
}

.typing-indicator span {
  height: 8px;
  width: 8px;
  background-color: #888;
  border-radius: 50%;
  display: inline-block;
  animation: bounce 1.3s infinite;
}

.typing-indicator span:nth-child(2) {
  animation-delay: 0.2s;
}

.typing-indicator span:nth-child(3) {
  animation-delay: 0.4s;
}

.input-form {
  display: flex;
  padding: 10px;
  background-color: white;
  border-top: 1px solid #ddd;
}

.input-form input {
  flex: 1;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 20px;
  margin-right: 10px;
  outline: none;
}

.input-form button {
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 10px 20px;
  cursor: pointer;
}

.sources-container {
  margin-top: 5px;
}

.sources-toggle {
  font-size: 12px;
  background: none;
  border: none;
  color: #007bff;
  cursor: pointer;
  padding: 0;
  text-decoration: underline;
}

.sources-content {
  display: none;
  margin-top: 5px;
  font-size: 12px;
  background-color: #f0f0f0;
  padding: 5px;
  border-radius: 5px;
}

.sources-container:hover .sources-content {
  display: block;
}

.source-item {
  margin-bottom: 5px;
  border-bottom: 1px solid #ddd;
  padding-bottom: 5px;
}

.source-file {
  font-style: italic;
  color: #666;
}

@keyframes bounce {
  0%, 60%, 100% { transform: translateY(0); }
  30% { transform: translateY(-5px); }
}

@media (max-width: 600px) {
  .chat-container {
    width: 100%;
    height: 100vh;
    border-radius: 0;
  }
}
```

### 5. CI/CD Script for Knowledge Base Updates

```bash
#!/bin/bash
# update_knowledge_base.sh - script for updating knowledge base

set -e

# Configuration
DOCS_DIR="/path/to/docs"
APP_DIR="/path/to/app"
BACKUP_DIR="/path/to/backups"
LOG_FILE="/var/log/knowledge_update.log"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "$(date) - Starting knowledge base update" >> $LOG_FILE

# Checking for document changes
CHECKSUM_FILE="$APP_DIR/docs_checksum.md5"
CURRENT_CHECKSUM=$(find $DOCS_DIR -type f -exec md5sum {} \; | sort | md5sum)

if [ -f $CHECKSUM_FILE ] && [ "$(cat $CHECKSUM_FILE)" == "$CURRENT_CHECKSUM" ]; then
    echo "$(date) - No changes in documents. Update skipped." >> $LOG_FILE
    exit 0
fi

# Backup current knowledge base
echo "$(date) - Creating backup of current knowledge base" >> $LOG_FILE
if [ -d "$APP_DIR/knowledge_base" ]; then
    mkdir -p $BACKUP_DIR
    tar -czf "$BACKUP_DIR/knowledge_base_$TIMESTAMP.tar.gz" -C $APP_DIR knowledge_base
fi

# Updating knowledge base
echo "$(date) - Running document indexing" >> $LOG_FILE
cd $APP_DIR
source venv/bin/activate
python indexer.py

# Saving new checksum
echo $CURRENT_CHECKSUM > $CHECKSUM_FILE

# Running regression tests
echo "$(date) - Running regression tests" >> $LOG_FILE
python tests/regression_tests.py

# Restarting application
echo "$(date) - Restarting application" >> $LOG_FILE
systemctl restart chatbot

echo "$(date) - Knowledge base update completed" >> $LOG_FILE
```

### 6. Docker Configuration

```dockerfile
# Dockerfile for chatbot backend
FROM python:3.9-slim

WORKDIR /app

# Installing dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copying application code
COPY . .

# Exposing port
EXPOSE 3000

# Running application
CMD ["gunicorn", "-b", "0.0.0.0:3000", "app:app", "--timeout", "120"]
```

```yaml
# docker-compose.yml
version: '3'

services:
  backend:
    build: ./backend
    ports:
      - "3000:3000"
    volumes:
      - ./knowledge_base:/app/knowledge_base
    environment:
      - PYTHONUNBUFFERED=1
    restart: unless-stopped
    
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped
```

### 7. Model Configuration for Different Hardware Variants

```python
# config.py - configuration dependent on available hardware

import os
import torch
import psutil

def get_config():
    """Returns configuration based on available hardware"""
    
    memory_gb = psutil.virtual_memory().total / (1024 ** 3)
    has_gpu = torch.cuda.is_available()
    num_cores = psutil.cpu_count(logical=False)
    
    config = {
        "use_local_model": False,
        "model_type": "api",
        "embedding_model": "sentence-transformers/paraphrase-multilingual-mpnet-base-v2",
        "chunk_size": 500,
        "retriever_k": 3,
    }
    
    # Configuration for weak hardware
    if memory_gb < 8:
        config["embedding_model"] = "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2"
        config["chunk_size"] = 300
        config["retriever_k"] = 2
    
    # Configuration for medium hardware without GPU
    elif memory_gb >= 8 and memory_gb < 16 and not has_gpu:
        config["use_local_model"] = True
        config["model_type"] = "llama.cpp"
        config["llm_model"] = "models/llama-2-7b-chat.Q4_K_M.gguf"
    
    # Configuration for powerful hardware with GPU
    elif has_gpu:
        config["use_local_model"] = True
        config["model_type"] = "transformers"
        config["llm_model"] = "mistralai/Mistral-7B-Instruct-v0.2"
        if torch.cuda.get_device_properties(0).total_memory > 8e9:  # over 8GB VRAM
            config["llm_model"] = "mistralai/Mistral-7B-Instruct-v0.2"
        else:
            config["llm_model"] = "microsoft/phi-2"
    
    return config
```

### 8. Integration with Existing PHP Application - Widget Embedding Script

```php
<?php
// chat_widget.php - script to embed in existing PHP application

/**
 * Generates HTML code for chatbot widget
 * 
 * @param string $userId User ID, if logged in
 * @param string $defaultLanguage Default chatbot language
 * @return string HTML code for widget
 */
function generateChatbotWidget($userId = null, $defaultLanguage = 'en') {
    // Generating token for authorization
    $token = '';
    if ($userId) {
        $secretKey = getenv('CHATBOT_SECRET_KEY') ?: 'default_secret_key';
        $token = hash_hmac('sha256', $userId, $secretKey);
    }
    
    $widgetUrl = "/chatbot/widget.js";
    $apiUrl = "/api/chatbot";
    
    $html = <<<HTML
    <div id="fiona-chatbot-container"></div>
    <script>
        window.FIONA_CHATBOT_CONFIG = {
            apiUrl: "{$apiUrl}",
            userId: "{$userId}",
            token: "{$token}",
            language: "{$defaultLanguage}"
        };
    </script>
    <script src="{$widgetUrl}"></script>
    HTML;
    
    return $html;
}

/**
 * API wrapper for chatbot - forwards query to backend and returns response
 */
function handleChatbotApiRequest() {
    // Checking if we are in REST API context
    if ($_SERVER['REQUEST_METHOD'] === 'POST' && strpos($_SERVER['REQUEST_URI'], '/api/chatbot') !== false) {
        header('Content-Type: application/json');
        
        // Getting input data
        $inputData = json_decode(file_get_contents('php://input'), true);
        if (!$inputData || !isset($inputData['query'])) {
            http_response_code(400);
            echo json_encode(['error' => 'Missing query']);
            exit;
        }
        
        // Token verification (optional)
        if (isset($inputData['userId']) && isset($inputData['token'])) {
            $secretKey = getenv('CHATBOT_SECRET_KEY') ?: 'default_secret_key';
            $expectedToken = hash_hmac('sha256', $inputData['userId'], $secretKey);
            
            if ($inputData['token'] !== $expectedToken) {
                http_response_code(401);
                echo json_encode(['error' => 'Invalid token']);
                exit;
            }
        }
        
        // Forwarding query to chatbot backend
        $ch = curl_init('http://localhost:3000/api/query');
        $data = json_encode([
            'query' => $inputData['query'],
            'language' => $inputData['language'] ?? 'en',
            'user_id' => $inputData['userId'] ?? 'anonymous'
        ]);
        
        curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        http_response_code($httpCode);
        echo $response;
        exit;
    }
}

// Automatic API handler invocation if we are in API context
handleChatbotApiRequest();
?>
```

## Summary

The proposed solution:

1. Uses RAG architecture to ensure response accuracy
2. Includes mechanisms to prevent hallucinations and verify sources
3. Offers CI/CD pipeline for knowledge base updates
4. Is scalable and deployable on various platforms
5. Has a modern, responsive user interface
6. Integrates with existing Apache2/PHP infrastructure

This approach will provide FIONA users with easy access to information about research project planning and Steve system features through an intuitive chatbot interface in multiple European languages.
