---
title: "Tích hợp Frontend (Next.js + FastAPI)"
date: 2024-01-01
weight: 14
chapter: false
pre: " <b> 5.14 </b> "
---

# Tích hợp Frontend: Next.js Dashboard + FastAPI Backend

Phần này bao gồm giao diện web cho News RAG system, bao gồm Dashboard, Search, AI Chat, Article Explorer, và Pipeline Monitor.

## Kiến trúc

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Browser   │────►│  Next.js     │────►│  FastAPI        │
│  (React)    │     │  (App Router)│     │  (Python)       │
└─────────────┘     └──────────────┘     └────────┬────────┘
                                                   │
                            ┌──────────────────────┼──────────────────────┐
                            ▼                      ▼                      ▼
                     ┌─────────────┐       ┌─────────────────┐    ┌─────────────┐
                     │  RAG API    │       │  Aurora DB      │    │  CloudWatch │
                     │  (Lambda)   │       │  (Direct SQL)   │    │  (Metrics)  │
                     └─────────────┘       └─────────────────┘    └─────────────┘
```

## Cấu trúc dự án

```
frontend/
├── nextjs-dashboard/          # Next.js 14+ App Router
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx              # Dashboard
│   │   │   ├── search/page.tsx       # Search page
│   │   │   ├── chat/page.tsx         # AI Chat
│   │   │   ├── explorer/page.tsx     # Article Explorer
│   │   │   └── monitor/page.tsx      # Pipeline Monitor
│   │   ├── components/
│   │   │   ├── ui/                   # Shadcn/UI components
│   │   │   ├── charts/               # Recharts wrappers
│   │   │   └── layout/
│   │   ├── lib/
│   │   │   ├── api.ts                # API client
│   │   │   └── utils.ts
│   │   └── hooks/
│   ├── package.json
│   ├── tailwind.config.ts
│   └── tsconfig.json
│
└── fastapi-backend/            # Optional: Python API for complex queries
    ├── main.py
    ├── routers/
    │   ├── stats.py
    │   ├── articles.py
    │   └── pipeline.py
    ├── database.py
    └── requirements.txt
```

## Next.js Dashboard Setup

### package.json
```json
{
  "name": "newsrag-dashboard",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "14.2.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "recharts": "^2.12.0",
    "date-fns": "^3.6.0",
    "lucide-react": "^0.372.0",
    "@radix-ui/react-slot": "^1.0.2",
    "@radix-ui/react-tabs": "^1.0.4",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-dropdown-menu": "^2.0.6",
    "@radix-ui/react-tooltip": "^1.0.7",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "swr": "^2.2.5",
    "zustand": "^4.5.0"
  },
  "devDependencies": {
    "@types/node": "^20.12.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "typescript": "^5.4.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.38",
    "autoprefixer": "^10.4.19",
    "eslint": "^8.57.0",
    "eslint-config-next": "14.2.0"
  }
}
```

### tailwind.config.ts
```ts
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./src/pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/components/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: "#f0f9ff",
          100: "#e0f2fe",
          500: "#0ea5e9",
          600: "#0284c7",
          700: "#0369a1",
        },
      },
    },
  },
  plugins: [],
};
export default config;
```

### API Client (lib/api.ts)
```typescript
const API_BASE = process.env.NEXT_PUBLIC_RAG_API_URL || "http://localhost:8000";
const FASTAPI_BASE = process.env.NEXT_PUBLIC_FASTAPI_URL || "http://localhost:8001";

export async function askRAG(query: string, model = "qwen3-8b-instant", topK = 5) {
  const res = await fetch(`${API_BASE}/ask`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ query, model, top_k: topK }),
  });
  if (!res.ok) throw new Error("RAG API error");
  return res.json();
}

export async function getStats() {
  const res = await fetch(`${FASTAPI_BASE}/stats/summary`);
  return res.json();
}

export async function getArticles(params: {
  source?: string;
  date_from?: string;
  date_to?: string;
  limit?: number;
  offset?: number;
}) {
  const search = new URLSearchParams(params as Record<string, string>).toString();
  const res = await fetch(`${FASTAPI_BASE}/articles?${search}`);
  return res.json();
}

export async function getPipelineStatus() {
  const res = await fetch(`${FASTAPI_BASE}/pipeline/status`);
  return res.json();
}
```

## Dashboard Page (app/page.tsx)

```tsx
"use client";

import { useEffect, useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { 
  BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer,
  PieChart, Pie, Cell, LineChart, Line
} from "recharts";
import { 
  Newspaper, Database, Zap, TrendingUp, Loader2 
} from "lucide-react";
import { getStats } from "@/lib/api";

const COLORS = ["#0ea5e9", "#22c55e", "#f59e0b", "#ef4444", "#8b5cf6", "#ec4899"];

export default function Dashboard() {
  const [stats, setStats] = useState<any>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    getStats().then(data => {
      setStats(data);
      setLoading(false);
    }).catch(() => setLoading(false));
  }, []);

  if (loading) return <div className="flex items-center justify-center h-64"><Loader2 className="h-8 w-8 animate-spin text-primary-600" /></div>;

  return (
    <div className="space-y-6 p-6">
      <h1 className="text-3xl font-bold">NewsRAG Dashboard</h1>

      {/* KPI Cards */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <StatCard title="Total Articles" value={stats?.total_articles || 0} icon={<Newspaper />} color="text-primary-600" />
        <StatCard title="Vector Chunks" value={stats?.total_chunks || 0} icon={<Database />} color="text-green-600" />
        <StatCard title="Sources" value={stats?.sources?.length || 0} icon={<Zap />} color="text-yellow-600" />
        <StatCard title="Avg Chunks/Article" value={stats?.avg_chunks_per_article || 0} icon={<TrendingUp />} color="text-purple-600" />
      </div>

      {/* Charts */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <Card>
          <CardHeader><CardTitle>Articles by Source</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={300}>
              <PieChart>
                <Pie
                  data={stats?.source_distribution || []}
                  cx="50%" cy="50%" innerRadius={60} outerRadius={100}
                  dataKey="count" nameKey="source_name" label={({ source_name, percent }) => `${source_name} ${(percent * 100).toFixed(0)}%`}
                >
                  {stats?.source_distribution?.map((_, i) => <Cell key={i} fill={COLORS[i % COLORS.length]} />)}
                </Pie>
                <Tooltip formatter={(v: number) => [v.toLocaleString(), "articles"]} />
              </PieChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>

        <Card>
          <CardHeader><CardTitle>Articles Over Time (30 days)</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={300}>
              <LineChart data={stats?.daily_counts || []}>
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis dataKey="date" tickFormatter={(v) => v.slice(5)} />
                <YAxis />
                <Tooltip />
                <Line type="monotone" dataKey="count" stroke="#0ea5e9" strokeWidth={2} dot={false} />
              </LineChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <Card>
          <CardHeader><CardTitle>Top Authors</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={300}>
              <BarChart data={stats?.top_authors || []} layout="vertical">
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis type="number" />
                <YAxis dataKey="author_name" type="category" width={150} />
                <Tooltip />
                <Bar dataKey="count" fill="#0ea5e9" radius={[0, 4, 4, 0]} />
              </BarChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>

        <Card>
          <CardHeader><CardTitle>Categories Distribution</CardTitle></CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={300}>
              <BarChart data={stats?.categories || []}>
                <CartesianGrid strokeDasharray="3 3" />
                <XAxis dataKey="category" tick={{ rotation: -45 }} />
                <YAxis />
                <Tooltip />
                <Bar dataKey="count" fill="#22c55e" />
              </BarChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}

function StatCard({ title, value, icon: Icon, color }: any) {
  return (
    <Card>
      <CardContent className="p-6">
        <div className="flex items-center justify-between">
          <div>
            <p className="text-sm text-muted-foreground">{title}</p>
            <p className="text-3xl font-bold">{value.toLocaleString()}</p>
          </div>
          <div className={`p-3 bg-primary-50 rounded-xl ${color}`}>
            <Icon className="h-6 w-6" />
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

## AI Chat Page (app/chat/page.tsx)

```tsx
"use client";

import { useState, useRef, useEffect } from "react";
import { useSWRConfig } from "swr";
import { askRAG } from "@/lib/api";
import { Button, Input, Textarea, Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui";
import { Send, Loader2, Bot, MessageSquare, Copy, Check } from "lucide-react";

const MODELS = [
  { id: "qwen3-8b-instant", name: "Qwen 3 8B (Groq)", provider: "Groq" },
  { id: "llama-3.1-8b-instant", name: "Llama 3.1 8B (Groq)", provider: "Groq" },
  { id: "gemini-2.0-flash", name: "Gemini 2.0 Flash", provider: "Google" },
];

interface Message {
  role: "user" | "assistant";
  content: string;
  sources?: any[];
  model?: string;
  duration_ms?: number;
}

export default function ChatPage() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");
  const [model, setModel] = useState("qwen3-8b-instant");
  const [loading, setLoading] = useState(false);
  const [copied, setCopied] = useState<string | null>(null);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const scrollToBottom = () => messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  useEffect(() => { scrollToBottom(); }, [messages]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!input.trim() || loading) return;

    const userMsg: Message = { role: "user", content: input };
    setMessages(prev => [...prev, userMsg]);
    setLoading(true);
    setInput("");

    try {
      const response = await askRAG(input, model, 5);
      const assistantMsg: Message = {
        role: "assistant",
        content: response.summary,
        sources: response.results,
        model: response.model,
        duration_ms: response.duration_ms,
      };
      setMessages(prev => [...prev, assistantMsg]);
    } catch (err) {
      setMessages(prev => [...prev, { 
        role: "assistant", 
        content: "❌ Error: Could not get response. Please try again." 
      }]);
    } finally {
      setLoading(false);
    }
  };

  const copyToClipboard = (text: string, id: string) => {
    navigator.clipboard.writeText(text);
    setCopied(id);
    setTimeout(() => setCopied(null), 2000);
  };

  return (
    <div className="h-[calc(100vh-8rem)] flex flex-col p-6 max-w-3xl mx-auto">
      <div className="mb-4 flex items-center gap-4">
        <h1">
          <Select value={model} onValueChange={setModel}>
            <SelectTrigger className="w-64">
              <SelectValue placeholder="Select model" />
            </SelectTrigger>
            <SelectContent>
              {MODELS.map(m => (
                <SelectItem key={m.id} value={m.id}>{m.name}</SelectItem>
              ))}
            </SelectContent>
          </Select>
        </div>

      <div className="flex-1 overflow-y-auto space-y-4" role="log" aria-live="polite">
        {messages.map((msg, idx) => (
          <div key={idx} className={`flex gap-3 ${msg.role === "user" ? "justify-end" : ""}`}>
            <div className={`max-w-[80%] ${msg.role === "user" ? "order-2" : ""}`}>
              <div className={`rounded-2xl px-4 py-3 ${msg.role === "user" ? "bg-primary-600 text-white" : "bg-gray-100"}`}>
                <p className="whitespace-pre-wrap">{msg.content}</p>
                {msg.sources && msg.sources.length > 0 && (
                  <details className="mt-2 text-sm">
                    <summary className="cursor-pointer text-muted-foreground">Sources ({msg.sources.length})</summary>
                    <ul className="mt-1 space-y-1">
                      {msg.sources.map((s: any, i: number) => (
                        <li key={i} className="text-muted-foreground">
                          [{i + 1}] {s.title} — {s.source_name} ({new Date(s.published_at).toLocaleDateString()})
                        </li>
                      ))}
                    </ul>
                  </details>
                )}
                {msg.duration_ms && (
                  <p className="mt-1 text-xs text-muted-foreground">
                    Model: {msg.model} • {msg.duration_ms.toFixed(0)}ms
                  </p>
                )}
              </div>
              {msg.role === "assistant" && (
                <button
                  onClick={() => copyToClipboard(msg.content, `msg-${idx}`)}
                  className="mt-1 text-xs text-muted-foreground hover:text-primary"
                >
                  {copied === `msg-${idx}` ? <Check className="inline h-3 w-3" /> : <Copy className="inline h-3 w-3" />}
                </button>
              )}
            </div>
            <div className={`flex-shrink-0 w-8 h-8 rounded-full flex items-center justify-center ${msg.role === "user" ? "bg-primary-100" : "bg-green-100"}`}>
              {msg.role === "user" ? <MessageSquare className="h-4 w-4 text-primary-600" /> : <Bot className="h-4 w-4 text-green-600" />}
            </div>
          </div>
        ))}
        <div ref={messagesEndRef} />
        {loading && (
          <div className="flex justify-start gap-3">
            <div className="w-8 h-8 rounded-full bg-green-100 flex items-center justify-center">
              <Loader2 className="h-4 w-4 text-green-600 animate-spin" />
            </div>
            <div className="bg-gray-100 rounded-2xl px-4 py-3 animate-pulse">
              <div className="h-4 bg-gray-200 rounded w-3/4 mb-2"></div>
              <div className="h-4 bg-gray-200 rounded w-1/2"></div>
            </div>
          </div>
        )}
      </div>

      <form onSubmit={handleSubmit} className="mt-4 flex gap-2">
        <Input
          value={input}
          onChange={e => setInput(e.target.value)}
          placeholder="Hỏi về tin tức..."
          disabled={loading}
          className="flex-1"
        />
        <Button type="submit" disabled={loading || !input.trim()} size="lg">
          <Send className="h-4 w-4" />
        </Button>
      </form>
    </div>
  );
}
```

## FastAPI Backend (Optional - cho direct DB queries)

```python
# fastapi-backend/main.py
from fastapi import FastAPI, Depends, Query
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import psycopg2
from psycopg2.extras import RealDictCursor
import os
from datetime import date, timedelta

app = FastAPI(title="NewsRAG Backend API")

app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

def get_db():
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST"),
        port=os.getenv("DB_PORT"),
        database=os.getenv("DB_NAME"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        cursor_factory=RealDictCursor,
    )
    try:
        yield conn
    finally:
        conn.close()

class StatsSummary(BaseModel):
    total_articles: int
    total_chunks: int
    sources: List[dict]
    daily_counts: List[dict]
    top_authors: List[dict]
    categories: List[dict]
    source_distribution: List[dict]
    avg_chunks_per_article: float

@app.get("/stats/summary", response_model=StatsSummary)
def get_stats(db=Depends(get_db)):
    with db.cursor() as cur:
        cur.execute("SELECT COUNT(*) as total FROM fact_articles")
        total_articles = cur.fetchone()["total"]

        cur.execute("SELECT COUNT(*) as total FROM fact_chunks WHERE embedding IS NOT NULL")
        total_chunks = cur.fetchone()["total"]

        cur.execute("SELECT source_name, source_domain, COUNT(*) as count FROM dim_source ds JOIN fact_articles fa ON ds.source_id = fa.source_id GROUP BY ds.source_id, ds.source_name, ds.source_domain ORDER BY count DESC")
        sources = cur.fetchall()

        cur.execute("""
            SELECT dt.date_day as date, COUNT(*) as count
            FROM fact_articles fa JOIN dim_time dt ON fa.time_id = dt.time_id
            WHERE dt.date_day >= (SELECT max(date_day) FROM dim_time) - INTERVAL '30 days'
            GROUP BY dt.date_day ORDER BY dt.date_day
        """)
        daily_counts = [{"date": str(r["date"]), "count": r["count"]} for r in cur.fetchall()]

        cur.execute("""
            SELECT da.author_name, COUNT(*) as count
            FROM fact_article_authors faa JOIN dim_author da ON faa.author_id = da.author_id
            GROUP BY da.author_id, da.author_name ORDER BY count DESC LIMIT 10
        """)
        top_authors = cur.fetchall()

        cur.execute("SELECT category, COUNT(*) as count FROM fact_articles WHERE category IS NOT NULL GROUP BY category ORDER BY count DESC LIMIT 10")
        categories = cur.fetchall()

        cur.execute("SELECT ds.source_name, COUNT(*) as count FROM fact_articles fa JOIN dim_source ds ON fa.source_id = ds.source_id GROUP BY ds.source_id, ds.source_name ORDER BY count DESC")
        source_dist = cur.fetchall()

        avg_chunks = total_chunks / total_articles if total_articles > 0 else 0

    return {
        "total_articles": total_articles,
        "total_chunks": total_chunks,
        "sources": sources,
        "daily_counts": daily_counts,
        "top_authors": top_authors,
        "categories": categories,
        "source_distribution": source_dist,
        "avg_chunks_per_article": round(avg_chunks, 1),
    }

class ArticleOut(BaseModel):
    article_id: int
    title: str
    source_name: str
    source_domain: str
    published_at: str
    category: str
    chunk_count: int

@app.get("/articles", response_model=List[ArticleOut])
def list_articles(
    source: Optional[str] = None,
    date_from: Optional[date] = None,
    date_to: Optional[date] = None,
    limit: int = 20,
    offset: int = 0,
    db=Depends(get_db)
):
    with db.cursor() as cur:
        sql = """
            SELECT fa.article_id, dc.title, ds.source_name, ds.source_domain,
                   dt.full_datetime as published_at, fa.category, fa.chunk_count
            FROM fact_articles fa
            JOIN dim_content dc ON fa.content_id = dc.content_id
            JOIN dim_source ds ON fa.source_id = ds.source_id
            JOIN dim_time dt ON fa.time_id = dt.time_id
            WHERE 1=1
        """
        params = []
        if source:
            sql += " AND ds.source_domain = %s"
            params.append(source)
        if date_from:
            sql += " AND dt.date_day >= %s"
            params.append(date_from)
        if date_to:
            sql += " AND dt.date_day <= %s"
            params.append(date_to)
        sql += " ORDER BY dt.full_datetime DESC LIMIT %s OFFSET %s"
        params.extend([limit, offset])
        cur.execute(sql, params)
        return cur.fetchall()

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

## Deployment

### Next.js on Vercel
```bash
# Connect GitHub repo to Vercel
# Add environment variables:
# NEXT_PUBLIC_RAG_API_URL=https://xxx.execute-api.ap-southeast-2.amazonaws.com/prod
# NEXT_PUBLIC_FASTAPI_URL=https://your-fastapi-url
```

### FastAPI on Cloud Run / ECS Fargate
```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# Build & push
docker build -t newsrag-fastapi .
docker tag newsrag-fastapi ${ECR_URI}/newsrag-fastapi:latest
docker push ${ECR_URI}/newsrag-fastapi:latest

# Deploy to ECS Fargate (similar to crawler task definition)
```

## Environment Variables

| Variable | Next.js | FastAPI | Description |
|----------|---------|---------|-------------|
| `NEXT_PUBLIC_RAG_API_URL` | ✅ | | Lambda RAG API endpoint |
| `NEXT_PUBLIC_FASTAPI_URL` | ✅ | | FastAPI backend URL |
| `DB_HOST` | | ✅ | Aurora endpoint |
| `DB_USER` / `DB_PASSWORD` | | ✅ | DB credentials |
| `GROQ_API_KEY` | | | For direct LLM calls if needed |

## Bước tiếp theo

Sau khi frontend hoạt động:
1. **Testing & Monitoring** - RAGAS evaluation: [Testing](5.15-Testing/)
2. **Cost Optimization** - Serverless tuning: [Cost](5.16-Cost/)
3. **Cleanup** - Remove resources: [Cleanup](5.17-Cleanup/)

---

**Tiếp theo:** [Kiểm thử & Giám sát](5.15-Testing/)