# MCP Client Web UI — Complete Project (Python FastAPI + React)

This canvas contains the full file contents for the MCP Client Web UI project you requested.

## Project structure

```
mcp-client-webui/
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── lll_client.py
│   │   ├── mcp_client.py
│   │   ├── schemas.py
│   │   ├── config.py
│   │   └── utils.py
│   ├── requirements.txt
│   └── start.sh
├── frontend/
│   ├── package.json
│   ├── vite.config.js
│   ├── index.html
│   ├── src/
│   │   ├── main.jsx
│   │   ├── App.jsx
│   │   ├── index.css
│   │   ├── api.js
│   │   └── components/
│   │       ├── Login.jsx
│   │       ├── Chat.jsx
│   │       ├── Sidebar.jsx
│   │       └── MessageBubble.jsx
│   └── README_FRONTEND.md
└── README.md
```

---

> **Important:** Do not store credentials in plaintext in production. Use Vault or short-lived tokens.

---

# Full file contents

---

## backend/app/main.py
```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from .config import settings
from .lll_client import LLMClient
from .mcp_client import MCPClient
from .schemas import ChatRequest, LoginRequest

app = FastAPI(title="MCP Client Host")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

llm_client = LLMClient()
mcp_client = MCPClient()

@app.post('/login')
async def login(req: LoginRequest):
    token = await llm_client.obtain_token(req.client_id, req.client_secret)
    if not token:
        raise HTTPException(status_code=401, detail='Auth failed')
    return {"token": token, "models": settings.DEFAULT_MODELS}

@app.post('/chat')
async def chat(payload: ChatRequest):
    token = payload.token
    model = payload.model
    messages = [m.dict() for m in payload.messages]

    # Ensure system prompt exists
    if not any(m['role'] == 'system' for m in messages):
        messages.insert(0, {"role": "system", "content": settings.SYSTEM_PROMPT})

    # 1) Call LLM
    resp = await llm_client.chat(model=model, messages=messages, bearer_token=token)
    if 'error' in resp:
        raise HTTPException(status_code=500, detail=resp['error'])

    text = resp.get('content', '')

    # 2) Try to parse JSON tool invocation
    import json
    parsed = None
    try:
        parsed = json.loads(text.strip())
    except Exception:
        parsed = None

    if parsed and isinstance(parsed, dict) and 'tool' in parsed:
        tool_name = parsed['tool']
        args = parsed.get('args', {})
        tool_result = await mcp_client.call_tool(tool_name, args)
        # send tool result back to LLM
        followup_messages = messages + [
            {"role": "assistant", "content": text},
            {"role": "tool", "name": tool_name, "content": str(tool_result)},
        ]
        final = await llm_client.chat(model=model, messages=followup_messages, bearer_token=token)
        if 'error' in final:
            raise HTTPException(status_code=500, detail=final['error'])
        return {"reply": final.get('content'), "tool_result": tool_result}

    return {"reply": text}
```

---

## backend/app/lll_client.py
```python
import base64
import aiohttp
from .config import settings

class LLMClient:
    def __init__(self):
        self.session = aiohttp.ClientSession()

    async def obtain_token(self, client_id: str, client_secret: str) -> str:
        # If AUTH_URL configured, do a token exchange. Otherwise fallback to base64
        if settings.AUTH_URL:
            async with self.session.post(settings.AUTH_URL, json={
                'client_id': client_id,
                'client_secret': client_secret,
                'grant_type': 'client_credentials',
            }) as r:
                if r.status == 200:
                    data = await r.json()
                    return data.get('access_token')
                else:
                    return None
        raw = f"{client_id}:{client_secret}".encode('utf-8')
        token = base64.b64encode(raw).decode('utf-8')
        return token

    async def chat(self, model: str, messages: list, bearer_token: str):
        url = settings.LLM_API_URL
        headers = {
            'Authorization': f'Bearer {bearer_token}',
            'Content-Type': 'application/json',
        }
        payload = {
            'model': model,
            'messages': messages,
            'temperature': 0.2,
        }
        async with self.session.post(url, json=payload, headers=headers) as r:
            if r.status != 200:
                text = await r.text()
                return {'error': f'LLM returned {r.status}: {text}'}
            data = await r.json()
            try:
                choice = data['choices'][0]['message']
                return {'content': choice['content']}
            except Exception:
                return {'content': str(data)}
```

---

## backend/app/mcp_client.py
```python
import aiohttp
from .config import settings

class MCPClient:
    def __init__(self):
        self.base = settings.MCP_SERVER_URL.rstrip('/')
        self.session = aiohttp.ClientSession()

    async def call_tool(self, tool_name: str, args: dict):
        # Default: POST {base}/tools/{tool_name}/invoke
        url = f"{self.base}/tools/{tool_name}/invoke"
        try:
            async with self.session.post(url, json=args) as r:
                text = await r.text()
                if r.status != 200:
                    return {'error': f'MCP error {r.status}: {text}'}
                try:
                    return await r.json()
                except Exception:
                    return {'result': text}
        except Exception as e:
            return {'error': str(e)}
```

---

## backend/app/schemas.py
```python
from pydantic import BaseModel
from typing import List

class LoginRequest(BaseModel):
    client_id: str
    client_secret: str

class Message(BaseModel):
    role: str
    content: str

class ChatRequest(BaseModel):
    token: str
    model: str
    messages: List[Message]
```

---

## backend/app/config.py
```python
from pydantic import BaseSettings

class Settings(BaseSettings):
    LLM_API_URL: str = "https://your-internal-llm.company.com/v1/chat/completions"
    AUTH_URL: str = ""  # optional token endpoint
    MCP_SERVER_URL: str = "http://localhost:8001"  # adjust to your MCP server
    DEFAULT_MODELS = ["company-model", "company-model-2", "general-1"]
    SYSTEM_PROMPT: str = (
        "You are an assistant with access to company tools via the MCP protocol. "
        "If you need to run a tool, reply with a single-line JSON object ONLY with this schema: "
        "{\"tool\": \"tool_name\", \"args\": { ... }}. Do not add any extra text. "
        "After the tool runs, you will receive the tool output and should then produce a human-readable answer."
    )

    class Config:
        env_file = '.env'

settings = Settings()
```

---

## backend/app/utils.py
```python
# small helpers (currently empty, placeholder)

```

---

## backend/requirements.txt
```
fastapi
uvicorn[standard]
aiohttp
pydantic
python-dotenv
```

---

## backend/start.sh
```bash
#!/usr/bin/env bash
# start backend (development)
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

---

# Frontend files

## frontend/package.json
```json
{
  "name": "mcp-client-webui",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.0.0",
    "vite": "^5.0.0",
    "tailwindcss": "^3.0.0",
    "postcss": "^8.0.0",
    "autoprefixer": "^10.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

---

## frontend/vite.config.js
```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: { port: 3000 }
})
```

---

## frontend/index.html
```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>MCP Client WebUI</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

---

## frontend/src/main.jsx
```jsx
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './index.css'

createRoot(document.getElementById('root')).render(<App />)
```

---

## frontend/src/App.jsx
```jsx
import React, { useState } from 'react'
import Login from './components/Login'
import Chat from './components/Chat'
import Sidebar from './components/Sidebar'

export default function App(){
  const [auth, setAuth] = useState(null)

  return (
    <div className="h-screen flex">
      <div className="w-64 border-r">
        <Sidebar />
      </div>
      <div className="flex-1 flex flex-col">
        {!auth ? (
          <Login onLogin={setAuth} />
        ) : (
          <Chat auth={auth} />
        )}
      </div>
    </div>
  )
}
```

---

## frontend/src/index.css
```css
html,body,#root{height:100%;}
body{font-family:Inter, ui-sans-serif, system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', Arial}

/* basic layout tweaks. Add Tailwind or keep CSS simple for now */

.container{display:flex;height:100%}
.sidebar{width:16rem;border-right:1px solid #e5e7eb}
.chat{flex:1;display:flex;flex-direction:column}

.message{padding:8px;margin:8px;border-radius:8px}
.message.user{background:#e6f4ff;align-self:flex-end}
.message.assistant{background:#f3f4f6;align-self:flex-start}
```

---

## frontend/src/api.js
```js
const API_BASE = import.meta.env.VITE_API_BASE || 'http://localhost:8000'

async function post(path, body){
  const res = await fetch(API_BASE + path, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  })
  return res.json()
}

export default { post }
```

---

## frontend/src/components/Login.jsx
```jsx
import React, { useState } from 'react'
import api from '../api'

export default function Login({ onLogin }){
  const [clientId, setClientId] = useState('')
  const [clientSecret, setClientSecret] = useState('')
  const [model, setModel] = useState('company-model')
  const [models, setModels] = useState(["company-model","company-model-2","general-1"])
  const [loading, setLoading] = useState(false)

  const submit = async () =>{
    setLoading(true)
    const res = await api.post('/login', { client_id: clientId, client_secret: clientSecret })
    setLoading(false)
    if(res.token){
      onLogin({ token: res.token, model })
    } else {
      alert('Login failed')
    }
  }

  return (
    <div className="p-6">
      <h2 className="text-xl font-bold mb-4">Login to your internal LLM</h2>
      <div className="mb-2">
        <input className="w-full p-2 border" placeholder="Client ID" value={clientId} onChange={e=>setClientId(e.target.value)} />
      </div>
      <div className="mb-2">
        <input type="password" className="w-full p-2 border" placeholder="Client Secret" value={clientSecret} onChange={e=>setClientSecret(e.target.value)} />
      </div>
      <div className="mb-4">
        <select className="w-full p-2 border" value={model} onChange={e=>setModel(e.target.value)}>
          {models.map(m=> <option key={m} value={m}>{m}</option>)}
        </select>
      </div>
      <div>
        <button onClick={submit} className="px-4 py-2 bg-blue-600 text-white rounded">{loading ? 'Logging in...' : 'Login'}</button>
      </div>
    </div>
  )
}
```

---

## frontend/src/components/Chat.jsx
```jsx
import React, { useState, useRef, useEffect } from 'react'
import api from '../api'
import MessageBubble from './MessageBubble'

export default function Chat({ auth }){
  const [messages, setMessages] = useState([])
  const [text, setText] = useState('')
  const scrollRef = useRef()

  useEffect(()=>{ if(scrollRef.current) scrollRef.current.scrollTop = scrollRef.current.scrollHeight }, [messages])

  const send = async () =>{
    if(!text.trim()) return
    const newMsg = { role: 'user', content: text }
    const payload = {
      token: auth.token,
      model: auth.model,
      messages: [...messages, newMsg]
    }
    setMessages(prev => [...prev, newMsg])
    setText('')
    const res = await api.post('/chat', payload)
    if(res.reply){
      setMessages(prev => [...prev, { role: 'assistant', content: res.reply }])
    } else {
      setMessages(prev => [...prev, { role: 'assistant', content: 'Error: no reply from server' }])
    }
  }

  return (
    <div className="flex-1 flex flex-col">
      <div ref={scrollRef} className="flex-1 overflow-auto p-4">
        {messages.map((m,i)=> <MessageBubble key={i} msg={m} />)}
      </div>
      <div className="p-4 border-t">
        <textarea rows={3} value={text} onChange={e=>setText(e.target.value)} className="w-full p-2 border" />
        <div className="mt-2 text-right">
          <button onClick={send} className="px-4 py-2 bg-green-600 text-white rounded">Send</button>
        </div>
      </div>
    </div>
  )
}
```

---

## frontend/src/components/MessageBubble.jsx
```jsx
import React from 'react'

export default function MessageBubble({ msg }){
  const cls = msg.role === 'user' ? 'message user' : 'message assistant'
  return (
    <div className={cls} style={{maxWidth:'70%'}}>
      <div>{msg.content}</div>
    </div>
  )
}
```

---

## frontend/src/components/Sidebar.jsx
```jsx
import React from 'react'

export default function Sidebar(){
  return (
    <div className="p-4">
      <h3 className="font-bold mb-2">Tools</h3>
      <ul>
        <li>get_logs</li>
        <li>query_db</li>
        <li>run_diagnostic</li>
      </ul>
    </div>
  )
}
```

---

## frontend/README_FRONTEND.md
```
1. npm install
2. npm run dev
3. edit VITE_API_BASE env var for backend URL
```

---

# README.md (root)
```markdown
# MCP Client WebUI

This project is a starter implementation of an MCP Host that uses your internal LLM (OpenAI-style API) and connects to an MCP server to call tools.

## How to run

### Backend

1. cd backend
2. python -m venv .venv
3. source .venv/bin/activate
4. pip install -r requirements.txt
5. create .env with values for LLM_API_URL, AUTH_URL (optional), MCP_SERVER_URL
6. ./start.sh

### Frontend

1. cd frontend
2. npm install
3. npm run dev

Open http://localhost:3000 and login with clientId and clientSecret, select model and start chatting.
```

---

If you'd like, I can:

- Add streaming support (server-sent events or WebSocket) for real-time LLM output.
- Add a `/tools` backend endpoint to fetch available tools from your MCP server and show them in the sidebar.
- Provide Dockerfiles for backend and frontend for easy deployment.

Tell me which extra features you want and I will add them to the project files.

