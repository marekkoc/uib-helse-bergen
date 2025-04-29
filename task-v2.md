# FIONA Chatbot Solution

## Project Overview

This document presents a comprehensive approach to creating a multilingual chatbot for the FIONA web application to help users access information from the "Plan your research project" and "Features of Steve" pages.

## 1. Architecture Solution

### 1.1 General Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Presentation Layer                          │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │  Frontend    │  │ UI Interface    │  │ PHP Integration    │  │
│  │  (React/Vue) │  │ (Modern UI)     │  │ (Apache2)          │  │
│  └──────────────┘  └─────────────────┘  └────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                         API Layer                               │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │  REST API    │  │ WebSocket       │  │ Authentication     │  │
│  │  (FastAPI)   │  │ (real-time)     │  │ (JWT/OAuth)        │  │
│  └──────────────┘  └─────────────────┘  └────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                     Processing Layer                            │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │  LLM Engine  │  │ Multilingual    │  │ Quality Control    │  │
│  │  (local)     │  │ (detection/     │  │ (anti-hallucination)  │
│  │              │  │  translation)   │  │                    │  │
│  └──────────────┘  └─────────────────┘  └────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                   Knowledge and Data Layer                      │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │  Vector DB   │  │ Document        │  │ Knowledge Update   │  │
│  │  (Chroma/    │  │ Processing      │  │ Monitoring System  │  │
│  │   Qdrant)    │  │ (pipeline)      │  │                    │  │
│  └──────────────┘  └─────────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Detailed Component Description

#### Knowledge and Data Layer:
- **Vector Database**: Chroma DB or Qdrant for storing document vectors
- **Document Indexing System**: Pipeline for processing HTML and PDF documents
- **Knowledge Update Monitoring**: System tracking changes in source documents

#### Processing Layer:
- **LLM Engine**: Locally hosted language model (e.g., Llama 3, Mistral, Gemma)
- **Anti-hallucination Module**: Response verification system
- **Multilingual Module**: Support for European languages

#### API Layer:
- **Backend**: FastAPI (Python) for high performance and easy ML integration
- **WebSockets**: For real-time communication
- **Integration Middleware**: For communication with existing PHP application

#### Presentation Layer:
- **Frontend**: Modern framework (React, Vue.js) with clean interface
- **Responsive Design**: Adaptation to mobile and desktop devices
- **PHP Integration**: Code for embedding chatbot in current application

## 2. Preventing Hallucinations and Providing Evidence

### 2.1 Retrieval-Augmented Generation (RAG)

The chatbot will use a RAG architecture, combined with additional methods to ensure reliability:

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ User Query    │────▶│ Document      │────▶│ Response      │
│               │     │ Retrieval     │     │ Generation    │
└───────────────┘     └───────────────┘     └───────────────┘
                             │                     │
                             ▼                     ▼
                      ┌───────────────┐     ┌───────────────┐
                      │ Vector DB     │     │ Fact          │
                      │ (embeddings)  │     │ Verification  │
                      └───────────────┘     └───────────────┘
```

### 2.2 Anti-hallucination Methods

1. **Confidence Vector Usage**:
   - Each knowledge fragment receives a confidence weight
   - Model only uses fragments with high confidence coefficient

2. **Fact Verification Module**:
   - Automatic verification of generated response against knowledge base
   - Detection of information without coverage in source documents

3. **Citations and References**:
   - Each response includes references to specific document fragments
   - User can click to see original context

### 2.3 User Interface with Evidence Example

```
User: How can I plan a new research project in FIONA?

Bot: To plan a new research project in FIONA, you need to:
1. Log into your account in the FIONA application [1]
2. Go to the "Plan your research project" section [1]
3. Fill out the project data form, including:
   - Project title
   - Research objectives
   - Required resources [2]
4. Submit the application for approval

[1] Source: Plan your research project, "Getting Started" section
[2] Source: Plan your research project, "Resource Requirements" section

[View Source #1] [View Source #2]
```

## 3. CI/CD Pipeline for Document Changes

### 3.1 CI/CD Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Document        │────▶│ Change          │────▶│ Change          │
│ Monitoring      │     │ Detection       │     │ Processing      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
┌─────────────────┐     ┌─────────────────┐            │
│ Knowledge Base  │◀────│ Quality         │◀───────────┘
│ Update          │     │ Testing         │
└─────────────────┘     └─────────────────┘
        │
        ▼
┌─────────────────┐
│ New Knowledge   │
│ Deployment      │
└─────────────────┘
```

### 3.2 CI/CD Workflow for Knowledge Updates

1. **Document Monitoring**:
   - GitLab/GitHub Actions scripts check for changes in documentation repositories
   - Cron job regularly checks website content for changes

2. **Change Detection and Processing**:
   - Comparison of document versions and identification of changed fragments
   - Automatic text extraction from new PDF/HTML documents
   - Chunk splitting, embedding generation

3. **Quality Testing**:
   - Automated tests checking responses to standard questions
   - Verification that new knowledge base correctly answers previous queries

4. **Update and Deployment**:
   - Vector store update with new embeddings
   - Deployment of new knowledge base without service interruption

### 3.3 Sample GitLab CI/CD Configuration

```yaml
stages:
  - detect_changes
  - process_content
  - test_knowledge
  - deploy

detect_document_changes:
  stage: detect_changes
  script:
    - python scripts/monitor_documents.py
  artifacts:
    paths:
      - changed_docs.json

process_document_content:
  stage: process_content
  script:
    - python scripts/process_documents.py
    - python scripts/generate_embeddings.py
  artifacts:
    paths:
      - new_embeddings/

test_knowledge_base:
  stage: test_knowledge
  script:
    - python scripts/test_question_answering.py
  dependencies:
    - process_document_content

deploy_knowledge_base:
  stage: deploy
  script:
    - python scripts/update_vector_db.py
  dependencies:
    - test_knowledge_base
  only:
    - master
```

## 4. Deployment on Different Hardware

### 4.1 Docker Containerization

The solution relies on Docker containerization, enabling easy deployment on various hardware configurations:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Docker Compose                             │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │ Frontend      │  │ Backend API   │  │ Vector DB     │        │
│  │ Container     │  │ Container     │  │ Container     │        │
│  └───────────────┘  └───────────────┘  └───────────────┘        │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │ LLM Model     │  │ Monitoring    │  │ Nginx/        │        │
│  │ Container     │  │ Container     │  │ Apache Proxy  │        │
│  └───────────────┘  └───────────────┘  └───────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Optimization for Different Hardware Configurations

#### 4.2.1 Low-resource Server Deployment
- Use of lighter LLM models (e.g., Llama 3 8B, Mistral 7B)
- Lower memory usage configuration, 4-bit model quantization
- Optional use of external NLP service APIs

#### 4.2.2 GPU Server Deployment
- GPU acceleration for faster responses
- Ability to use larger models (e.g., Llama 3 70B, Mistral-Large)
- Support for more parallel queries

#### 4.2.3 Server Cluster Deployment
- Distribution of microservices across multiple machines
- Horizontal scaling for handling more traffic
- Load balancing between instances

### 4.3 Automatic Deployment Script

```bash
#!/bin/bash
# deploy.sh - FIONA Chatbot automatic deployment script

# Detect available hardware resources
MEMORY=$(free -g | grep Mem | awk '{print $2}')
CPU_CORES=$(nproc)
HAS_GPU=$(lspci | grep -i nvidia | wc -l)

# Select appropriate configuration
if [ $HAS_GPU -gt 0 ]; then
  echo "GPU detected - using GPU-accelerated configuration"
  CONFIG_FILE="docker-compose.gpu.yml"
elif [ $MEMORY -lt 16 ]; then
  echo "Limited memory detected - using lightweight configuration"
  CONFIG_FILE="docker-compose.light.yml"
else
  echo "Using standard CPU configuration"
  CONFIG_FILE="docker-compose.standard.yml"
fi

# Deployment with selected configuration
docker-compose -f $CONFIG_FILE up -d
```

## 5. User Interface

### 5.1 UI Design

Modern, responsive user interface with the following features:

- Minimalist design with clear typography
- Dark/light mode support
- Animated typing indicators for better interaction experience
- Highlighted citations and source references

### 5.2 Real-time Interaction

- WebSockets for real-time communication
- Character-by-character response streaming
- Immediate suggestions while typing queries

### 5.3 UI Features Enhancing Usability

- Conversation history with ability to return to previous questions
- Response rating capability (system feedback)
- "View Source" button for each response fragment
- Expandable technical explanations

### 5.4 Sample React Code for User Interface

```jsx
// ChatInterface.jsx
import React, { useState, useEffect, useRef } from 'react';
import { useWebSocket } from '../hooks/useWebSocket';
import ChatMessage from './ChatMessage';
import SourceViewer from './SourceViewer';

const ChatInterface = () => {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [isTyping, setIsTyping] = useState(false);
  const endOfMessagesRef = useRef(null);
  
  const { sendMessage, lastMessage } = useWebSocket('ws://localhost:8000/ws/chat');
  
  useEffect(() => {
    if (lastMessage) {
      const data = JSON.parse(lastMessage.data);
      if (data.type === 'typing_start') {
        setIsTyping(true);
      } else if (data.type === 'message') {
        setIsTyping(false);
        setMessages(prev => [...prev, {
          id: Date.now(),
          sender: 'bot',
          text: data.text,
          sources: data.sources
        }]);
      }
    }
  }, [lastMessage]);
  
  useEffect(() => {
    scrollToBottom();
  }, [messages, isTyping]);
  
  const scrollToBottom = () => {
    endOfMessagesRef.current?.scrollIntoView({ behavior: 'smooth' });
  };
  
  const handleSubmit = (e) => {
    e.preventDefault();
    if (!input.trim()) return;
    
    const userMessage = {
      id: Date.now(),
      sender: 'user',
      text: input
    };
    
    setMessages(prev => [...prev, userMessage]);
    sendMessage(JSON.stringify({
      type: 'message',
      text: input
    }));
    
    setInput('');
  };
  
  return (
    <div className="chat-container">
      <div className="chat-messages">
        {messages.map(msg => (
          <ChatMessage 
            key={msg.id} 
            message={msg} 
          />
        ))}
        {isTyping && (
          <div className="typing-indicator">
            <span></span>
            <span></span>
            <span></span>
          </div>
        )}
        <div ref={endOfMessagesRef} />
      </div>
      
      <form onSubmit={handleSubmit} className="chat-input-form">
        <input
          type="text"
          value={input}
          onChange={e => setInput(e.target.value)}
          placeholder="Ask about FIONA..."
          className="chat-input"
        />
        <button type="submit" className="send-button">
          Send
        </button>
      </form>
    </div>
  );
};

export default ChatInterface;
```

## 6. Integration with Existing Application (Apache2 + PHP 8.x)

### 6.1 Integration Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                   Existing PHP Application                     │
│                                                               │
│  ┌───────────────┐           ┌───────────────────────────┐    │
│  │ Apache2       │           │                           │    │
│  │ (front-end)   │◀────┐     │ PHP 8.x application       │    │
│  └───────────────┘     │     │                           │    │
│         │              │     └───────────────────────────┘    │
│         ▼              │                   │                  │
│  ┌───────────────┐     │                   ▼                  │
│  │ mod_proxy     │     │     ┌───────────────────────────┐    │
│  │ mod_rewrite   │─────┘     │ PHP API Connector         │    │
│  └───────────────┘           └───────────────────────────┘    │
└───────────────────────────────────┬───────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                  FIONA Chatbot (new application)              │
│  ┌───────────────┐           ┌───────────────────────────┐    │
│  │ FastAPI       │           │                           │    │
│  │ Backend       │◀──────────│ LLM Engine + Vector DB    │    │
│  └───────────────┘           │                           │    │
└───────────────────────────────────────────────────────────────┘
```

### 6.2 Integration Methods

#### 6.2.1 Apache Server Level Integration

Using Apache modules:
- `mod_proxy` to redirect requests to chatbot
- `mod_rewrite` for transparent URL integration

```apache
# Sample Apache configuration (chatbot.conf)
<VirtualHost *:80>
    ServerName fiona.example.com
    DocumentRoot /var/www/html/fiona

    # Chatbot request redirection
    ProxyPass /api/chatbot http://localhost:8000
    ProxyPassReverse /api/chatbot http://localhost:8000

    # Serving static chatbot files
    Alias /chatbot /var/www/html/chatbot/static
    <Directory /var/www/html/chatbot/static>
        Require all granted
    </Directory>

    # Standard PHP configuration
    <FilesMatch \.php$>
        SetHandler application/x-httpd-php
    </FilesMatch>
</VirtualHost>
```

#### 6.2.2 PHP Level Integration

PHP component for easy integration with existing application:

```php
<?php
// FionaChatbotConnector.php
namespace Fiona\Chatbot;

class ChatbotConnector {
    private $apiUrl;
    private $apiKey;
    
    public function __construct($apiUrl = 'http://localhost:8000', $apiKey = null) {
        $this->apiUrl = $apiUrl;
        $this->apiKey = $apiKey;
    }
    
    public function renderChatWidget($containerId = 'fiona-chatbot', $options = []) {
        $defaultOptions = [
            'theme' => 'light',
            'position' => 'bottom-right',
            'width' => '350px',
            'height' => '500px',
            'language' => 'auto'
        ];
        
        $mergedOptions = array_merge($defaultOptions, $options);
        $optionsJson = json_encode($mergedOptions);
        
        $html = <<<HTML
        <div id="{$containerId}"></div>
        <script src="/chatbot/js/fiona-chatbot.js"></script>
        <script>
            document.addEventListener('DOMContentLoaded', function() {
                FionaChatbot.init('{$containerId}', {
                    apiUrl: '{$this->apiUrl}',
                    apiKey: '{$this->apiKey}',
                    options: {$optionsJson}
                });
            });
        </script>
        HTML;
        
        return $html;
    }
    
    public function askQuestion($question, $conversationId = null, $language = 'auto') {
        $curl = curl_init();
        curl_setopt_array($curl, [
            CURLOPT_URL => "{$this->apiUrl}/api/ask",
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([
                'question' => $question,
                'conversation_id' => $conversationId,
                'language' => $language
            ]),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: Bearer ' . $this->apiKey
            ]
        ]);
        
        $response = curl_exec($curl);
        curl_close($curl);
        
        return json_decode($response, true);
    }
}
?>
```

#### 6.2.3 Usage Example in Existing PHP Application

```php
<?php
// Anywhere in existing PHP application
require_once 'vendor/autoload.php';

use Fiona\Chatbot\ChatbotConnector;

// Connector initialization
$chatbot = new ChatbotConnector('http://localhost:8000', 'api_key_here');

// Chatbot widget rendering in appropriate page section
echo $chatbot->renderChatWidget('fiona-chat-container', [
    'theme' => 'dark',
    'language' => 'en'
]);
?>
```

## 7. Technical Advantages of the Solution

1. **Multilingual Support**:
   - Full support for European languages using multilingual LLM models
   - Automatic query language detection
   - Response translation to query language

2. **Fact-based Responses**:
   - RAG system with source citation for every piece of information
   - Response verification mechanisms before sending to user
   - Transparency through ability to check sources

3. **Deployment Flexibility**:
   - Containerization allowing deployment in various environments
   - Automatic adaptation to available hardware resources
   - Simple integration with existing infrastructure

4. **Modern and Responsive UI**:
   - User-friendly interface with animations enhancing UX
   - Immediate response through WebSockets
   - Intuitive navigation and source access

5. **Easy Knowledge Base Updates**:
   - Automatic processing of document changes
   - CI/CD pipeline ensuring knowledge consistency
   - Quality tests for new knowledge base versions

## 8. Technologies

### Backend:
- Python 3.10+
- FastAPI
- LlamaIndex/LangChain (RAG framework)
- Llama 3 / Mistral (local LLM model)
- ChromaDB / Qdrant (vector store)
- WebSockets (asyncio)

### Frontend:
- React / Vue.js
- TypeScript
- TailwindCSS
- WebSocket API

### DevOps:
- Docker + Docker Compose
- GitLab CI/CD
- Ansible (deployment automation)
- Prometheus + Grafana (monitoring)

### Integration:
- Apache mod_proxy
- PHP 8.x Connector Library
