# Criando um Agente Autônomo com LLM usando Jido

**Objetivo:** Ao final deste guia, você saberá como criar um agente autônomo que interage com um LLM (Large Language Model) usando a arquitetura funcional pura do Jido.

## Visão Geral da Arquitetura

O Jido segue o padrão **Signal → Action → cmd/2 → {agent, directives}**. Para um agente LLM, o fluxo é:

```
Usuário envia mensagem
    → Signal chega no AgentServer
    → Agent processa via cmd/2
    → Action chama a API do LLM
    → Resposta atualiza o estado do agent
    → Directives emitem eventos (se necessário)
```

## Pré-requisitos

Adicione as dependências no `mix.exs`:

```elixir
defp deps do
  [
    {:jido, "~> 2.0"},
    {:jido_ai, "~> 0.1"},     # Integração com LLMs
    {:req_llm, "~> 0.1"},     # Cliente HTTP para APIs de LLM
    {:req, "~> 0.5"}          # Cliente HTTP base
  ]
end
```

## Passo 1: Definir as Actions

Actions são transformações puras de estado. Para um agente LLM, precisamos de actions para enviar mensagens e processar respostas.

### Action para Chamar o LLM

```elixir
defmodule MyApp.Actions.ChatWithLLM do
  use Jido.Action,
    name: "chat_with_llm",
    description: "Envia uma mensagem ao LLM e retorna a resposta",
    schema: [
      message: [type: :string, required: true],
      model: [type: :string, default: "claude-sonnet-4-20250514"],
      system_prompt: [type: :string, default: "Você é um assistente útil."]
    ]

  @impl true
  def run(params, context) do
    messages = Map.get(context.state, :messages, [])

    # Monta o histórico de conversa + nova mensagem
    conversation =
      messages ++ [%{role: "user", content: params.message}]

    # Chama a API do LLM (usando req_llm ou Req direto)
    case call_llm(params.model, params.system_prompt, conversation) do
      {:ok, response} ->
        updated_messages =
          conversation ++ [%{role: "assistant", content: response}]

        {:ok, %{
          messages: updated_messages,
          last_response: response,
          status: :idle
        }}

      {:error, reason} ->
        {:ok, %{status: :error, last_error: reason}}
    end
  end

  defp call_llm(model, system_prompt, messages) do
    # Implementação com req_llm ou chamada HTTP direta
    # Exemplo com Req:
    Req.post("https://api.anthropic.com/v1/messages",
      json: %{
        model: model,
        system: system_prompt,
        messages: messages,
        max_tokens: 1024
      },
      headers: [
        {"x-api-key", System.get_env("ANTHROPIC_API_KEY")},
        {"anthropic-version", "2023-06-01"}
      ]
    )
    |> case do
      {:ok, %{status: 200, body: body}} ->
        content = body["content"] |> List.first() |> Map.get("text")
        {:ok, content}
      {:ok, %{status: status, body: body}} ->
        {:error, "API error #{status}: #{inspect(body)}"}
      {:error, reason} ->
        {:error, reason}
    end
  end
end
```

### Action para Limpar Conversa

```elixir
defmodule MyApp.Actions.ClearConversation do
  use Jido.Action,
    name: "clear_conversation",
    description: "Limpa o histórico de conversa",
    schema: []

  @impl true
  def run(_params, _context) do
    {:ok, %{messages: [], last_response: nil, status: :idle}}
  end
end
```

## Passo 2: Definir o Plugin de Chat

Encapsule as actions e o estado em um Plugin reutilizável:

```elixir
defmodule MyApp.LLMChatPlugin do
  use Jido.Plugin,
    name: "llm_chat",
    state_key: :chat,
    actions: [
      MyApp.Actions.ChatWithLLM,
      MyApp.Actions.ClearConversation
    ],
    schema: Zoi.object(%{
      messages: Zoi.list(Zoi.any()) |> Zoi.default([]),
      last_response: Zoi.any() |> Zoi.default(nil),
      model: Zoi.string() |> Zoi.default("claude-sonnet-4-20250514"),
      system_prompt: Zoi.string() |> Zoi.default("Você é um assistente útil."),
      status: Zoi.atom() |> Zoi.default(:idle)
    }),
    signal_routes: [
      {"chat.send", MyApp.Actions.ChatWithLLM},
      {"chat.clear", MyApp.Actions.ClearConversation}
    ]

  @impl Jido.Plugin
  def mount(_agent, config) do
    {:ok, %{
      model: Map.get(config, :model, "claude-sonnet-4-20250514"),
      system_prompt: Map.get(config, :system_prompt, "Você é um assistente útil.")
    }}
  end
end
```

## Passo 3: Definir o Agente

```elixir
defmodule MyApp.LLMAgent do
  use Jido.Agent,
    name: "llm_agent",
    description: "Agente autônomo com capacidade de conversa via LLM",
    schema: [
      status: [type: :atom, default: :idle],
      task_count: [type: :integer, default: 0]
    ],
    plugins: [
      {MyApp.LLMChatPlugin, %{
        model: "claude-sonnet-4-20250514",
        system_prompt: "Você é um assistente autônomo. Analise tarefas e responda de forma objetiva."
      }}
    ],
    signal_routes: [
      {"chat.send", MyApp.Actions.ChatWithLLM},
      {"chat.clear", MyApp.Actions.ClearConversation}
    ]
end
```

## Passo 4: Uso Puro (sem processos)

Teste a lógica do agente de forma pura, sem precisar de GenServer:

```elixir
# Criar o agente
agent = MyApp.LLMAgent.new()

# Enviar uma mensagem ao LLM
{agent, _directives} = MyApp.LLMAgent.cmd(agent, {
  MyApp.Actions.ChatWithLLM,
  %{message: "Explique o que é Elixir em uma frase."}
})

# A resposta está no estado
IO.puts(agent.state.last_response)

# Continuar a conversa (o histórico é mantido)
{agent, _directives} = MyApp.LLMAgent.cmd(agent, {
  MyApp.Actions.ChatWithLLM,
  %{message: "E quais são suas principais vantagens?"}
})

IO.puts(agent.state.last_response)

# Limpar conversa
{agent, []} = MyApp.LLMAgent.cmd(agent, MyApp.Actions.ClearConversation)
```

## Passo 5: Uso em Produção com AgentServer

Para aplicações reais, execute o agente dentro do AgentServer:

```elixir
# Defina a instância Jido
defmodule MyApp.Jido do
  use Jido, otp_app: :my_app
end

# No application.ex
children = [MyApp.Jido]
Supervisor.start_link(children, strategy: :one_for_one)

# Inicie o agente
{:ok, pid} = MyApp.Jido.start_agent(MyApp.LLMAgent, id: "assistant-1")

# Envie signals de forma síncrona
signal = Jido.Signal.new!("chat.send", %{message: "Olá, como você pode me ajudar?"}, source: "/user")
{:ok, agent} = Jido.AgentServer.call(pid, signal)
IO.puts(agent.state.last_response)

# Ou de forma assíncrona
signal = Jido.Signal.new!("chat.send", %{message: "Liste 3 benefícios do Elixir"}, source: "/user")
:ok = Jido.AgentServer.cast(pid, signal)
```

## Passo 6: Agente Autônomo com FSM

Para um agente verdadeiramente autônomo que raciocina em etapas (ex: padrão ReAct), use a estratégia FSM:

```elixir
defmodule MyApp.AutonomousAgent do
  use Jido.Agent,
    name: "autonomous_agent",
    description: "Agente autônomo que raciocina e age em ciclos",
    strategy: {Jido.Agent.Strategy.FSM,
      initial_state: "idle",
      transitions: %{
        "idle" => ["thinking"],
        "thinking" => ["acting", "idle", "completed"],
        "acting" => ["thinking", "completed"],
        "completed" => ["idle"]
      }
    },
    plugins: [
      {MyApp.LLMChatPlugin, %{
        system_prompt: """
        Você é um agente autônomo. Para cada tarefa:
        1. PENSE sobre o que precisa ser feito
        2. AJA executando a próxima etapa
        3. OBSERVE o resultado
        4. Repita até completar a tarefa

        Responda em JSON com o formato:
        {"thought": "...", "action": "...", "is_complete": true/false}
        """
      }}
    ],
    schema: [
      current_task: [type: :string, default: ""],
      steps_taken: [type: :integer, default: 0],
      max_steps: [type: :integer, default: 10]
    ]
end
```

### Action de Raciocínio (Think-Act-Observe)

```elixir
defmodule MyApp.Actions.ReasoningStep do
  use Jido.Action,
    name: "reasoning_step",
    description: "Executa um passo de raciocínio autônomo",
    schema: [
      task: [type: :string, required: true]
    ]

  @impl true
  def run(params, context) do
    steps = Map.get(context.state, :steps_taken, 0)
    max = Map.get(context.state, :max_steps, 10)

    if steps >= max do
      {:ok, %{status: :max_steps_reached}}
    else
      # Chama o LLM para decidir o próximo passo
      case reason_about_task(params.task, context.state) do
        {:ok, %{"is_complete" => true} = result} ->
          {:ok, %{
            last_response: result["thought"],
            steps_taken: steps + 1,
            status: :completed
          }}

        {:ok, result} ->
          {:ok, %{
            last_response: result["thought"],
            steps_taken: steps + 1,
            current_task: params.task
          },
          # Emite diretiva para continuar o ciclo
          [%Jido.Agent.Directive.Emit{
            signal_type: "agent.continue",
            data: %{task: params.task}
          }]}

        {:error, reason} ->
          {:ok, %{status: :error, last_error: reason}}
      end
    end
  end

  defp reason_about_task(task, state) do
    # Implementação da chamada ao LLM com contexto do estado atual
    # Retorna o JSON parseado com thought/action/is_complete
    # ...
  end
end
```

## Passo 7: Usando Thread e Memory

Os plugins padrão Thread e Memory dão ao agente "memória" persistente:

```elixir
alias Jido.Memory.Agent, as: MemoryAgent
alias Jido.Thread

# Inicializar memória
agent = MyApp.LLMAgent.new()
agent = MemoryAgent.ensure(agent)

# Armazenar fatos no espaço "world"
agent = MemoryAgent.put_in_space(agent, :world, :user_name, "João")
agent = MemoryAgent.put_in_space(agent, :world, :preference, "respostas curtas")

# Adicionar tarefas
agent = MemoryAgent.append_to_space(agent, :tasks, %{
  id: "t1",
  text: "Resumir o documento X",
  status: :pending
})

# O agente pode consultar sua memória durante o raciocínio
user_name = MemoryAgent.get_in_space(agent, :world, :user_name)
```

## Padrões Avançados

### Multi-Agente: Agente Orquestrador

```elixir
defmodule MyApp.OrchestratorAgent do
  use Jido.Agent,
    name: "orchestrator",
    description: "Coordena múltiplos agentes especializados",
    schema: [
      child_agents: [type: {:list, :string}, default: []]
    ]
end

# O orquestrador pode criar agentes filhos via directives
{agent, directives} = MyApp.OrchestratorAgent.cmd(agent, {
  Jido.Actions.Lifecycle.SpawnChild,
  %{
    module: MyApp.LLMAgent,
    id: "researcher-1",
    config: %{system_prompt: "Você é um pesquisador."}
  }
})
# directives conterá um SpawnAgent para o runtime executar
```

### Sensor para Eventos Externos

```elixir
defmodule MyApp.WebhookSensor do
  use Jido.Sensor,
    name: "webhook_sensor",
    description: "Recebe webhooks e alimenta o agente",
    schema: Zoi.object(%{
      endpoint: Zoi.string()
    })

  @impl true
  def init(config, _ctx) do
    {:ok, %{endpoint: config.endpoint}, []}
  end

  @impl true
  def handle_event({:webhook, data}, state) do
    signal = Jido.Signal.new!("webhook.received", data, source: "/webhook")
    {:ok, state, [{:emit, signal}]}
  end
end
```

## Resumo

| Conceito | Descrição |
|----------|-----------|
| **Agent** | Struct imutável com estado e schema |
| **Action** | Transformação pura de estado (onde a chamada ao LLM acontece) |
| **Plugin** | Empacota actions + estado isolado (ex: chat, memória) |
| **Strategy** | Padrão de execução (Direct para simples, FSM para autônomo) |
| **Signal** | Mensagem CloudEvents para comunicação |
| **Directive** | Efeito colateral descrito como dado (emit, spawn, schedule) |
| **Thread** | Log append-only da conversa |
| **Memory** | Crenças e tarefas do agente |
| **Sensor** | Fonte de eventos externos |

## Próximos Passos

- [Getting Started](getting-started.livemd) — Quickstart de 5 minutos
- [Core Loop](core-loop.md) — Entenda o modelo mental
- [Plugins](plugins.md) — Composição de capacidades
- [Strategies](strategies.md) — Padrões de execução (Direct, FSM)
- [Directives](directives.md) — Efeitos colaterais
- [jido_ai docs](https://hexdocs.pm/jido_ai) — Integração completa com LLMs
