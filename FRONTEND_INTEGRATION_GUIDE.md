# AutoGen 前端集成开发指南

## 快速概览

AutoGen Studio 提供了完整的前端-后端集成解决方案，前端开发者可以通过以下方式集成：

1. **直接使用 AutoGen Studio**: 完整的 Web 应用
2. **API 集成**: 通过 REST API 集成到现有应用
3. **组件嵌入**: 使用 iframe 或 Web Components
4. **自定义开发**: 基于 AutoGen 框架构建自定义前端

## 核心 API 接口文档

### 认证系统

```typescript
// 登录接口
POST /auth/login
{
  "username": "string",
  "password": "string"
}

// 响应
{
  "status": true,
  "data": {
    "token": "jwt_token_here",
    "user": {
      "id": "user_id",
      "username": "username",
      "email": "email@example.com"
    }
  }
}

// 获取用户信息
GET /auth/user
Headers: { "Authorization": "Bearer <token>" }
```

### 团队管理 API

```typescript
// 获取用户的所有团队
GET /api/teams?user_id={user_id}
{
  "status": true,
  "data": [
    {
      "id": 1,
      "name": "我的AI团队",
      "description": "包含助手和分析师的团队",
      "user_id": "user_123",
      "component": {
        "name": "team_config",
        "agents": [...],
        "workflow": "round_robin",
        "termination_condition": {...}
      },
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ]
}

// 创建新团队
POST /api/teams
{
  "name": "客服团队",
  "user_id": "user_123",
  "component": {
    "name": "customer_service_team",
    "agents": [
      {
        "name": "客服助手",
        "type": "assistant_agent",
        "config": {
          "model_client": {
            "provider": "openai",
            "model": "gpt-4",
            "api_key": "..."
          },
          "system_message": "你是一个友好的客服助手"
        }
      }
    ],
    "workflow": "sequential",
    "termination_condition": {
      "type": "max_messages",
      "config": { "max_messages": 20 }
    }
  }
}
```

### 会话管理 API

```typescript
// 创建会话
POST /api/sessions
{
  "team_id": 1,
  "user_id": "user_123",
  "name": "客户咨询-001"
}

// 响应
{
  "status": true,
  "data": {
    "id": "session_123",
    "team_id": 1,
    "user_id": "user_123",
    "status": "active",
    "created_at": "2024-01-01T00:00:00Z"
  }
}

// 获取会话列表
GET /api/sessions?user_id={user_id}&team_id={team_id}

// 发送消息
POST /api/sessions/{session_id}/messages
{
  "content": "你好，我需要帮助",
  "type": "text",
  "source": "user"
}
```

### WebSocket 实时通信

```typescript
// 连接 WebSocket
WS /ws/{session_id}

// 消息格式
{
  "type": "message",
  "data": {
    "id": "msg_123",
    "session_id": "session_123",
    "source": "assistant",
    "content": "你好！我是AI助手，很高兴为您服务。",
    "timestamp": "2024-01-01T00:00:00Z",
    "metadata": {
      "model_used": "gpt-4",
      "tokens_used": 15
    }
  }
}

// 状态更新
{
  "type": "status",
  "data": {
    "session_id": "session_123",
    "status": "processing",
    "agent": "客服助手"
  }
}

// 错误消息
{
  "type": "error",
  "data": {
    "message": "API 调用失败",
    "code": "api_error"
  }
}
```

## React 集成示例

### 1. AutoGen API 客户端封装

```typescript
// autoGenClient.ts
export interface AutoGenConfig {
  baseURL: string;
  apiKey?: string;
  timeout?: number;
}

export interface Agent {
  name: string;
  type: 'assistant_agent' | 'user_proxy_agent' | 'custom_agent';
  config: {
    model_client?: ModelConfig;
    system_message?: string;
    tools?: Tool[];
  };
}

export interface ModelConfig {
  provider: 'openai' | 'azure' | 'anthropic' | 'ollama';
  model: string;
  api_key?: string;
  base_url?: string;
  temperature?: number;
  max_tokens?: number;
}

export interface Team {
  id?: number;
  name: string;
  user_id: string;
  component: {
    name: string;
    agents: Agent[];
    workflow: 'sequential' | 'round_robin' | 'group_chat';
    termination_condition?: TerminationCondition;
  };
}

export interface Session {
  id: string;
  team_id: number;
  user_id: string;
  status: 'active' | 'completed' | 'error';
  created_at: string;
  updated_at: string;
}

export interface Message {
  id: string;
  session_id: string;
  source: string;
  content: string;
  timestamp: string;
  metadata?: Record<string, any>;
}

export class AutoGenClient {
  private config: AutoGenConfig;
  private wsConnection?: WebSocket;

  constructor(config: AutoGenConfig) {
    this.config = config;
  }

  // 团队管理
  async getTeams(userId: string): Promise<Team[]> {
    const response = await this.request('GET', `/api/teams?user_id=${userId}`);
    return response.data;
  }

  async createTeam(team: Omit<Team, 'id'>): Promise<Team> {
    const response = await this.request('POST', '/api/teams', team);
    return response.data;
  }

  async updateTeam(teamId: number, team: Partial<Team>): Promise<Team> {
    const response = await this.request('PUT', `/api/teams/${teamId}`, team);
    return response.data;
  }

  async deleteTeam(teamId: number, userId: string): Promise<void> {
    await this.request('DELETE', `/api/teams/${teamId}?user_id=${userId}`);
  }

  // 会话管理
  async createSession(teamId: number, userId: string, name?: string): Promise<Session> {
    const response = await this.request('POST', '/api/sessions', {
      team_id: teamId,
      user_id: userId,
      name
    });
    return response.data;
  }

  async getSessions(userId: string, teamId?: number): Promise<Session[]> {
    const params = new URLSearchParams({ user_id: userId });
    if (teamId) params.set('team_id', teamId.toString());
    
    const response = await this.request('GET', `/api/sessions?${params}`);
    return response.data;
  }

  async sendMessage(sessionId: string, content: string, type: string = 'text'): Promise<void> {
    await this.request('POST', `/api/sessions/${sessionId}/messages`, {
      content,
      type,
      source: 'user'
    });
  }

  // WebSocket 连接
  connectWebSocket(
    sessionId: string,
    onMessage: (message: any) => void,
    onError?: (error: Event) => void,
    onClose?: (event: CloseEvent) => void
  ): void {
    const wsUrl = this.config.baseURL.replace(/^http/, 'ws') + `/ws/${sessionId}`;
    this.wsConnection = new WebSocket(wsUrl);

    this.wsConnection.onopen = () => {
      console.log('WebSocket connected');
    };

    this.wsConnection.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        onMessage(data);
      } catch (error) {
        console.error('Failed to parse WebSocket message:', error);
      }
    };

    this.wsConnection.onerror = (error) => {
      console.error('WebSocket error:', error);
      onError?.(error);
    };

    this.wsConnection.onclose = (event) => {
      console.log('WebSocket disconnected');
      onClose?.(event);
    };
  }

  disconnectWebSocket(): void {
    if (this.wsConnection) {
      this.wsConnection.close();
      this.wsConnection = undefined;
    }
  }

  // 私有方法
  private async request(method: string, path: string, body?: any): Promise<any> {
    const url = `${this.config.baseURL}${path}`;
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
    };

    if (this.config.apiKey) {
      headers.Authorization = `Bearer ${this.config.apiKey}`;
    }

    const response = await fetch(url, {
      method,
      headers,
      body: body ? JSON.stringify(body) : undefined,
      signal: AbortSignal.timeout(this.config.timeout || 30000),
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return response.json();
  }
}
```

### 2. React Hooks 封装

```typescript
// hooks/useAutoGen.ts
import { useState, useEffect, useCallback } from 'react';
import { AutoGenClient, Team, Session, Message } from '../services/autoGenClient';

export const useAutoGen = (config: { baseURL: string; apiKey?: string }) => {
  const [client] = useState(() => new AutoGenClient(config));
  const [teams, setTeams] = useState<Team[]>([]);
  const [sessions, setSessions] = useState<Session[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const loadTeams = useCallback(async (userId: string) => {
    setLoading(true);
    setError(null);
    try {
      const teamsData = await client.getTeams(userId);
      setTeams(teamsData);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load teams');
    } finally {
      setLoading(false);
    }
  }, [client]);

  const createTeam = useCallback(async (team: Omit<Team, 'id'>) => {
    setLoading(true);
    setError(null);
    try {
      const newTeam = await client.createTeam(team);
      setTeams(prev => [...prev, newTeam]);
      return newTeam;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to create team');
      throw err;
    } finally {
      setLoading(false);
    }
  }, [client]);

  const loadSessions = useCallback(async (userId: string, teamId?: number) => {
    setLoading(true);
    setError(null);
    try {
      const sessionsData = await client.getSessions(userId, teamId);
      setSessions(sessionsData);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load sessions');
    } finally {
      setLoading(false);
    }
  }, [client]);

  return {
    client,
    teams,
    sessions,
    loading,
    error,
    loadTeams,
    createTeam,
    loadSessions,
  };
};

// hooks/useWebSocket.ts
import { useState, useEffect, useRef } from 'react';
import { AutoGenClient, Message } from '../services/autoGenClient';

export const useWebSocket = (client: AutoGenClient, sessionId: string | null) => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [connectionStatus, setConnectionStatus] = useState<'disconnected' | 'connecting' | 'connected'>('disconnected');
  const messagesRef = useRef<Message[]>([]);

  useEffect(() => {
    if (!sessionId) return;

    setConnectionStatus('connecting');
    
    client.connectWebSocket(
      sessionId,
      (data) => {
        if (data.type === 'message') {
          const newMessage = data.data as Message;
          messagesRef.current = [...messagesRef.current, newMessage];
          setMessages([...messagesRef.current]);
        }
        setConnectionStatus('connected');
      },
      (error) => {
        console.error('WebSocket error:', error);
        setConnectionStatus('disconnected');
      },
      () => {
        setConnectionStatus('disconnected');
      }
    );

    return () => {
      client.disconnectWebSocket();
      setConnectionStatus('disconnected');
    };
  }, [client, sessionId]);

  const sendMessage = useCallback(async (content: string) => {
    if (!sessionId) return;
    
    try {
      await client.sendMessage(sessionId, content);
      // 添加用户消息到本地状态
      const userMessage: Message = {
        id: `user_${Date.now()}`,
        session_id: sessionId,
        source: 'user',
        content,
        timestamp: new Date().toISOString(),
      };
      messagesRef.current = [...messagesRef.current, userMessage];
      setMessages([...messagesRef.current]);
    } catch (error) {
      console.error('Failed to send message:', error);
    }
  }, [client, sessionId]);

  return {
    messages,
    connectionStatus,
    sendMessage,
  };
};
```

### 3. 完整的聊天组件

```typescript
// components/ChatInterface.tsx
import React, { useState, useRef, useEffect } from 'react';
import { useAutoGen } from '../hooks/useAutoGen';
import { useWebSocket } from '../hooks/useWebSocket';

interface ChatInterfaceProps {
  teamId: number;
  userId: string;
  className?: string;
}

export const ChatInterface: React.FC<ChatInterfaceProps> = ({
  teamId,
  userId,
  className = '',
}) => {
  const { client, loading, error } = useAutoGen({
    baseURL: process.env.REACT_APP_AUTOGEN_API_URL || 'http://localhost:8080',
  });

  const [session, setSession] = useState<Session | null>(null);
  const [input, setInput] = useState('');
  const messagesEndRef = useRef<HTMLDivElement>(null);

  const { messages, connectionStatus, sendMessage } = useWebSocket(
    client,
    session?.id || null
  );

  // 创建会话
  useEffect(() => {
    const initSession = async () => {
      try {
        const newSession = await client.createSession(teamId, userId);
        setSession(newSession);
      } catch (err) {
        console.error('Failed to create session:', err);
      }
    };

    initSession();
  }, [client, teamId, userId]);

  // 自动滚动到底部
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const handleSendMessage = async () => {
    if (!input.trim() || !session) return;

    await sendMessage(input);
    setInput('');
  };

  const handleKeyPress = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleSendMessage();
    }
  };

  if (loading) {
    return (
      <div className={`flex items-center justify-center h-64 ${className}`}>
        <div className="text-gray-500">正在加载...</div>
      </div>
    );
  }

  if (error) {
    return (
      <div className={`flex items-center justify-center h-64 ${className}`}>
        <div className="text-red-500">错误: {error}</div>
      </div>
    );
  }

  return (
    <div className={`flex flex-col h-full ${className}`}>
      {/* 连接状态指示器 */}
      <div className="flex items-center px-4 py-2 bg-gray-50 border-b">
        <div className={`w-2 h-2 rounded-full mr-2 ${
          connectionStatus === 'connected' ? 'bg-green-500' : 
          connectionStatus === 'connecting' ? 'bg-yellow-500' : 'bg-red-500'
        }`} />
        <span className="text-sm text-gray-600">
          {connectionStatus === 'connected' ? '已连接' : 
           connectionStatus === 'connecting' ? '连接中...' : '未连接'}
        </span>
      </div>

      {/* 消息列表 */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.length === 0 ? (
          <div className="text-center text-gray-500 mt-8">
            开始与AI智能体对话吧！
          </div>
        ) : (
          messages.map((message) => (
            <MessageBubble key={message.id} message={message} />
          ))
        )}
        <div ref={messagesEndRef} />
      </div>

      {/* 输入区域 */}
      <div className="border-t p-4">
        <div className="flex space-x-2">
          <textarea
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyPress={handleKeyPress}
            placeholder="输入消息... (Shift+Enter 换行)"
            className="flex-1 p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 resize-none"
            rows={1}
            style={{ maxHeight: '120px' }}
          />
          <button
            onClick={handleSendMessage}
            disabled={!input.trim() || connectionStatus !== 'connected'}
            className="px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
          >
            发送
          </button>
        </div>
      </div>
    </div>
  );
};

// 消息气泡组件
const MessageBubble: React.FC<{ message: Message }> = ({ message }) => {
  const isUser = message.source === 'user';
  
  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'}`}>
      <div className={`max-w-xs lg:max-w-md px-4 py-2 rounded-lg ${
        isUser 
          ? 'bg-blue-500 text-white' 
          : 'bg-gray-200 text-gray-800'
      }`}>
        {!isUser && (
          <div className="text-xs font-semibold mb-1 opacity-75">
            {message.source}
          </div>
        )}
        <div className="text-sm whitespace-pre-wrap">
          {message.content}
        </div>
        <div className={`text-xs mt-1 opacity-75 ${
          isUser ? 'text-blue-100' : 'text-gray-500'
        }`}>
          {new Date(message.timestamp).toLocaleTimeString()}
        </div>
      </div>
    </div>
  );
};
```

### 4. 团队管理组件

```typescript
// components/TeamManager.tsx
import React, { useState, useEffect } from 'react';
import { useAutoGen } from '../hooks/useAutoGen';
import { Team, Agent } from '../services/autoGenClient';

interface TeamManagerProps {
  userId: string;
  onTeamSelect: (team: Team) => void;
}

export const TeamManager: React.FC<TeamManagerProps> = ({ userId, onTeamSelect }) => {
  const { teams, loading, error, loadTeams, createTeam } = useAutoGen({
    baseURL: process.env.REACT_APP_AUTOGEN_API_URL || 'http://localhost:8080',
  });

  const [showCreateForm, setShowCreateForm] = useState(false);

  useEffect(() => {
    loadTeams(userId);
  }, [loadTeams, userId]);

  if (loading) {
    return <div className="text-center py-8">正在加载团队...</div>;
  }

  if (error) {
    return <div className="text-center py-8 text-red-500">错误: {error}</div>;
  }

  return (
    <div className="p-6">
      <div className="flex justify-between items-center mb-6">
        <h2 className="text-2xl font-bold">AI 智能体团队</h2>
        <button
          onClick={() => setShowCreateForm(true)}
          className="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition-colors"
        >
          创建新团队
        </button>
      </div>

      {teams.length === 0 ? (
        <div className="text-center py-12 text-gray-500">
          <p>还没有创建任何团队</p>
          <button
            onClick={() => setShowCreateForm(true)}
            className="mt-4 px-6 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition-colors"
          >
            创建第一个团队
          </button>
        </div>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {teams.map((team) => (
            <TeamCard 
              key={team.id} 
              team={team} 
              onSelect={() => onTeamSelect(team)} 
            />
          ))}
        </div>
      )}

      {showCreateForm && (
        <CreateTeamModal
          userId={userId}
          onClose={() => setShowCreateForm(false)}
          onCreate={async (teamData) => {
            await createTeam(teamData);
            setShowCreateForm(false);
          }}
        />
      )}
    </div>
  );
};

// 团队卡片组件
const TeamCard: React.FC<{ team: Team; onSelect: () => void }> = ({ team, onSelect }) => {
  return (
    <div className="bg-white border rounded-lg p-6 hover:shadow-lg transition-shadow cursor-pointer" onClick={onSelect}>
      <h3 className="text-lg font-semibold mb-2">{team.name}</h3>
      <p className="text-gray-600 text-sm mb-4">
        {team.component.agents.length} 个智能体
      </p>
      <div className="space-y-2">
        {team.component.agents.slice(0, 3).map((agent, index) => (
          <div key={index} className="flex items-center text-sm">
            <div className="w-2 h-2 bg-green-500 rounded-full mr-2" />
            <span>{agent.name}</span>
          </div>
        ))}
        {team.component.agents.length > 3 && (
          <div className="text-sm text-gray-500">
            还有 {team.component.agents.length - 3} 个智能体...
          </div>
        )}
      </div>
    </div>
  );
};

// 创建团队模态框
const CreateTeamModal: React.FC<{
  userId: string;
  onClose: () => void;
  onCreate: (team: Omit<Team, 'id'>) => Promise<void>;
}> = ({ userId, onClose, onCreate }) => {
  const [teamName, setTeamName] = useState('');
  const [agents, setAgents] = useState<Agent[]>([
    {
      name: '助手',
      type: 'assistant_agent',
      config: {
        system_message: '你是一个有用的AI助手。',
      },
    },
  ]);
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!teamName.trim()) return;

    setLoading(true);
    try {
      await onCreate({
        name: teamName,
        user_id: userId,
        component: {
          name: teamName.toLowerCase().replace(/\s+/g, '_'),
          agents,
          workflow: 'round_robin',
        },
      });
    } catch (error) {
      console.error('Failed to create team:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg p-6 w-full max-w-md">
        <h3 className="text-lg font-semibold mb-4">创建新团队</h3>
        
        <form onSubmit={handleSubmit}>
          <div className="mb-4">
            <label className="block text-sm font-medium mb-2">团队名称</label>
            <input
              type="text"
              value={teamName}
              onChange={(e) => setTeamName(e.target.value)}
              className="w-full p-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
              placeholder="例如：客服团队"
              required
            />
          </div>

          <div className="mb-6">
            <label className="block text-sm font-medium mb-2">智能体配置</label>
            {agents.map((agent, index) => (
              <div key={index} className="border rounded-lg p-3 mb-2">
                <input
                  type="text"
                  value={agent.name}
                  onChange={(e) => {
                    const newAgents = [...agents];
                    newAgents[index].name = e.target.value;
                    setAgents(newAgents);
                  }}
                  className="w-full p-2 border border-gray-300 rounded mb-2"
                  placeholder="智能体名称"
                />
                <textarea
                  value={agent.config.system_message || ''}
                  onChange={(e) => {
                    const newAgents = [...agents];
                    newAgents[index].config.system_message = e.target.value;
                    setAgents(newAgents);
                  }}
                  className="w-full p-2 border border-gray-300 rounded"
                  placeholder="系统提示词"
                  rows={2}
                />
              </div>
            ))}
            
            <button
              type="button"
              onClick={() => setAgents([...agents, {
                name: `智能体 ${agents.length + 1}`,
                type: 'assistant_agent',
                config: { system_message: '' },
              }])}
              className="text-blue-500 text-sm hover:text-blue-600"
            >
              + 添加智能体
            </button>
          </div>

          <div className="flex space-x-3">
            <button
              type="button"
              onClick={onClose}
              className="flex-1 px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors"
            >
              取消
            </button>
            <button
              type="submit"
              disabled={loading || !teamName.trim()}
              className="flex-1 px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:opacity-50 transition-colors"
            >
              {loading ? '创建中...' : '创建'}
            </button>
          </div>
        </form>
      </div>
    </div>
  );
};
```

## 部署配置

### 环境变量设置

```bash
# .env
REACT_APP_AUTOGEN_API_URL=http://localhost:8080
REACT_APP_WS_URL=ws://localhost:8080

# 生产环境
REACT_APP_AUTOGEN_API_URL=https://api.yourapp.com
REACT_APP_WS_URL=wss://api.yourapp.com
```

### Docker 部署示例

```yaml
# docker-compose.yml
version: '3.8'
services:
  autogen-backend:
    image: autogen-studio:latest
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/autogen
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    depends_on:
      - db

  frontend:
    build: .
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_AUTOGEN_API_URL=http://autogen-backend:8080
    depends_on:
      - autogen-backend

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=autogen
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

这个指南提供了完整的前端集成方案，包括 API 封装、React Hooks、组件示例和部署配置，帮助前端开发者快速集成 AutoGen 功能到现有应用中。