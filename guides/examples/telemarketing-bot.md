# Exemplo: Robô de Telemarketing com Jido

Este guia mostra como implementar um robô de telemarketing autônomo usando Jido com integração LLM. O robô segue um fluxo de conversa estruturado via FSM (máquina de estados finitos) e usa um LLM para gerar respostas naturais.

## Arquitetura

```
                    ┌──────────────────────────────┐
                    │      TelemarketingAgent       │
                    │  FSM: greeting → pitch →      │
                    │  objection → closing → done   │
                    └──────────────┬───────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                     │
     ┌────────▼────────┐  ┌───────▼────────┐  ┌────────▼────────┐
     │  LLM Chat Plugin │  │ Call Log Plugin │  │  Memory Plugin  │
     │  (gera respostas)│  │ (registra tudo) │  │ (dados cliente) │
     └─────────────────┘  └────────────────┘  └─────────────────┘
```

## Passo 1: Actions do Fluxo de Telemarketing

### Iniciar Ligação (Saudação)

```elixir
defmodule MyApp.Telemarketing.Actions.StartCall do
  use Jido.Action,
    name: "start_call",
    description: "Inicia a ligação com saudação personalizada",
    schema: [
      customer_name: [type: :string, required: true],
      customer_phone: [type: :string, required: true],
      product: [type: :string, required: true]
    ]

  @impl true
  def run(params, _context) do
    {:ok, %{
      customer_name: params.customer_name,
      customer_phone: params.customer_phone,
      product: params.product,
      call_started_at: DateTime.utc_now(),
      phase: :greeting,
      messages: [
        %{role: "system", content: system_prompt(params)},
        %{role: "assistant", content: greeting(params.customer_name)}
      ],
      attempts: 0,
      outcome: nil
    }}
  end

  defp system_prompt(params) do
    """
    Você é um operador de telemarketing profissional e educado.
    Produto: #{params.product}.
    Cliente: #{params.customer_name}.

    Regras:
    - Seja sempre cordial e respeitoso
    - Nunca seja agressivo ou insistente demais
    - Se o cliente pedir para parar, encerre educadamente
    - Destaque benefícios, não características
    - Limite-se a no máximo 2 tentativas de contornar objeções
    - Respostas curtas e naturais (máximo 3 frases)
    """
  end

  defp greeting(name) do
    "Bom dia, #{name}! Aqui é da empresa XYZ. Tudo bem com você? " <>
    "Estou ligando porque temos uma oportunidade especial que pode te interessar."
  end
end
```

### Fazer o Pitch do Produto

```elixir
defmodule MyApp.Telemarketing.Actions.DeliverPitch do
  use Jido.Action,
    name: "deliver_pitch",
    description: "Apresenta o produto ao cliente",
    schema: []

  @impl true
  def run(_params, context) do
    product = context.state[:product] || "nosso produto"
    name = context.state[:customer_name] || "cliente"

    pitch = """
    #{name}, nós temos o #{product} que está com uma condição especial este mês. \
    Os clientes que aderiram estão muito satisfeitos com os resultados. \
    Posso te explicar rapidamente como funciona?
    """

    messages = Map.get(context.state, :messages, [])
    updated = messages ++ [%{role: "assistant", content: String.trim(pitch)}]

    {:ok, %{messages: updated, phase: :pitch}}
  end
end
```

### Processar Resposta do Cliente (via LLM)

```elixir
defmodule MyApp.Telemarketing.Actions.ProcessResponse do
  use Jido.Action,
    name: "process_response",
    description: "Processa a resposta do cliente usando LLM para gerar resposta contextual",
    schema: [
      customer_message: [type: :string, required: true]
    ]

  alias Jido.Agent.Directive

  @impl true
  def run(params, context) do
    messages = Map.get(context.state, :messages, [])
    attempts = Map.get(context.state, :attempts, 0)

    # Adiciona a mensagem do cliente ao histórico
    messages = messages ++ [%{role: "user", content: params.customer_message}]

    # Classifica a intenção do cliente
    intent = classify_intent(params.customer_message)

    case intent do
      :interested ->
        response = "Que ótimo! Fico feliz com seu interesse. Deixa eu te passar os detalhes..."
        messages = messages ++ [%{role: "assistant", content: response}]
        {:ok, %{messages: messages, phase: :closing, intent: :interested}}

      :objection when attempts < 2 ->
        # Usa LLM para gerar resposta à objeção
        case generate_objection_response(messages) do
          {:ok, response} ->
            messages = messages ++ [%{role: "assistant", content: response}]
            {:ok, %{messages: messages, phase: :objection, attempts: attempts + 1}}

          {:error, _reason} ->
            fallback = "Entendo sua preocupação. Posso te enviar mais informações por email?"
            messages = messages ++ [%{role: "assistant", content: fallback}]
            {:ok, %{messages: messages, phase: :objection, attempts: attempts + 1}}
        end

      :objection ->
        # Limite de tentativas atingido, encerrar educadamente
        response = "Entendo perfeitamente, #{context.state[:customer_name]}. " <>
          "Agradeço muito seu tempo. Caso mude de ideia, estaremos à disposição!"
        messages = messages ++ [%{role: "assistant", content: response}]
        {:ok, %{messages: messages, phase: :done, outcome: :declined}}

      :stop ->
        response = "Claro, desculpe pelo incômodo. Tenha um ótimo dia!"
        messages = messages ++ [%{role: "assistant", content: response}]
        {:ok, %{messages: messages, phase: :done, outcome: :stopped}}

      :question ->
        case generate_answer(messages) do
          {:ok, response} ->
            messages = messages ++ [%{role: "assistant", content: response}]
            {:ok, %{messages: messages, phase: context.state[:phase]}}

          {:error, _} ->
            response = "Boa pergunta! Vou verificar isso e te retorno em seguida."
            messages = messages ++ [%{role: "assistant", content: response}]
            {:ok, %{messages: messages, phase: context.state[:phase]}}
        end

      _other ->
        case generate_contextual_response(messages) do
          {:ok, response} ->
            messages = messages ++ [%{role: "assistant", content: response}]
            {:ok, %{messages: messages}}

          {:error, _} ->
            {:ok, %{messages: messages}}
        end
    end
  end

  # Classificação simples de intenção — em produção use o LLM
  defp classify_intent(message) do
    msg = String.downcase(message)

    cond do
      Regex.match?(~r/sim|claro|quero|interessado|pode falar|me conta/, msg) ->
        :interested
      Regex.match?(~r/não|caro|sem tempo|depois|agora não|não preciso/, msg) ->
        :objection
      Regex.match?(~r/para|pare|não ligue|desist|tchau|encerr/, msg) ->
        :stop
      Regex.match?(~r/como|quanto|qual|onde|quando|por que|\?/, msg) ->
        :question
      true ->
        :neutral
    end
  end

  defp generate_objection_response(messages) do
    call_llm(messages ++ [
      %{role: "system", content:
        "O cliente fez uma objeção. Responda de forma empática, " <>
        "reconheça a preocupação e apresente um contra-argumento sutil. " <>
        "Máximo 2 frases."}
    ])
  end

  defp generate_answer(messages) do
    call_llm(messages ++ [
      %{role: "system", content:
        "O cliente fez uma pergunta. Responda de forma clara e objetiva. " <>
        "Máximo 2 frases."}
    ])
  end

  defp generate_contextual_response(messages) do
    call_llm(messages ++ [
      %{role: "system", content:
        "Continue a conversa de forma natural. Máximo 2 frases."}
    ])
  end

  defp call_llm(messages) do
    # Filtra mensagens de sistema para o formato da API
    api_messages =
      messages
      |> Enum.reject(&(&1.role == "system"))

    system =
      messages
      |> Enum.filter(&(&1.role == "system"))
      |> Enum.map(& &1.content)
      |> Enum.join("\n\n")

    case Req.post("https://api.anthropic.com/v1/messages",
      json: %{
        model: "claude-sonnet-4-20250514",
        system: system,
        messages: api_messages,
        max_tokens: 256
      },
      headers: [
        {"x-api-key", System.get_env("ANTHROPIC_API_KEY")},
        {"anthropic-version", "2023-06-01"}
      ]
    ) do
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

### Fechar a Venda

```elixir
defmodule MyApp.Telemarketing.Actions.CloseCall do
  use Jido.Action,
    name: "close_call",
    description: "Encerra a ligação e registra o resultado",
    schema: [
      outcome: [type: :atom, default: nil]
    ]

  alias Jido.Agent.Directive

  @impl true
  def run(params, context) do
    outcome = params.outcome || context.state[:outcome] || :unknown
    name = context.state[:customer_name] || "cliente"
    messages = Map.get(context.state, :messages, [])

    farewell =
      case outcome do
        :sale_closed ->
          "Perfeito, #{name}! Vou finalizar seu cadastro. " <>
          "Você vai receber uma confirmação por SMS. Muito obrigado!"

        :declined ->
          "Sem problemas, #{name}. Agradeço seu tempo. Tenha um ótimo dia!"

        :stopped ->
          "Desculpe pelo incômodo. Tenha um bom dia!"

        :callback_scheduled ->
          "Combinado, #{name}! Vou te ligar novamente no horário combinado. Até mais!"

        _ ->
          "Agradeço sua atenção, #{name}. Tenha um bom dia!"
      end

    messages = messages ++ [%{role: "assistant", content: farewell}]

    # Emite sinal com resultado da ligação para CRM/analytics
    signal = Jido.Signal.new!(
      "call.completed",
      %{
        customer_name: context.state[:customer_name],
        customer_phone: context.state[:customer_phone],
        product: context.state[:product],
        outcome: outcome,
        duration_messages: length(messages),
        completed_at: DateTime.utc_now()
      },
      source: "/telemarketing"
    )

    {:ok,
      %{
        messages: messages,
        phase: :done,
        outcome: outcome,
        call_ended_at: DateTime.utc_now()
      },
      [Directive.emit(signal)]
    }
  end
end
```

### Agendar Callback

```elixir
defmodule MyApp.Telemarketing.Actions.ScheduleCallback do
  use Jido.Action,
    name: "schedule_callback",
    description: "Agenda retorno de ligação",
    schema: [
      callback_time: [type: :string, required: true],
      notes: [type: :string, default: ""]
    ]

  alias Jido.Agent.Directive

  @impl true
  def run(params, context) do
    messages = Map.get(context.state, :messages, [])
    response = "Perfeito! Anotei aqui: #{params.callback_time}. Vou te ligar nesse horário."
    messages = messages ++ [%{role: "assistant", content: response}]

    signal = Jido.Signal.new!(
      "callback.scheduled",
      %{
        customer_name: context.state[:customer_name],
        customer_phone: context.state[:customer_phone],
        callback_time: params.callback_time,
        notes: params.notes,
        product: context.state[:product]
      },
      source: "/telemarketing"
    )

    {:ok,
      %{messages: messages, phase: :done, outcome: :callback_scheduled},
      [Directive.emit(signal)]
    }
  end
end
```

## Passo 2: Plugin de Telemarketing

```elixir
defmodule MyApp.Telemarketing.Plugin do
  use Jido.Plugin,
    name: "telemarketing",
    state_key: :telemarketing,
    description: "Plugin de telemarketing com fluxo de conversa e integração LLM",
    actions: [
      MyApp.Telemarketing.Actions.StartCall,
      MyApp.Telemarketing.Actions.DeliverPitch,
      MyApp.Telemarketing.Actions.ProcessResponse,
      MyApp.Telemarketing.Actions.CloseCall,
      MyApp.Telemarketing.Actions.ScheduleCallback
    ],
    schema: Zoi.object(%{
      messages: Zoi.list(Zoi.any()) |> Zoi.default([]),
      customer_name: Zoi.string() |> Zoi.default(""),
      customer_phone: Zoi.string() |> Zoi.default(""),
      product: Zoi.string() |> Zoi.default(""),
      phase: Zoi.atom() |> Zoi.default(:idle),
      outcome: Zoi.any() |> Zoi.default(nil),
      attempts: Zoi.integer() |> Zoi.default(0),
      intent: Zoi.any() |> Zoi.default(nil),
      call_started_at: Zoi.any() |> Zoi.default(nil),
      call_ended_at: Zoi.any() |> Zoi.default(nil)
    }),
    signal_routes: [
      {"call.start", MyApp.Telemarketing.Actions.StartCall},
      {"call.pitch", MyApp.Telemarketing.Actions.DeliverPitch},
      {"call.response", MyApp.Telemarketing.Actions.ProcessResponse},
      {"call.close", MyApp.Telemarketing.Actions.CloseCall},
      {"call.schedule_callback", MyApp.Telemarketing.Actions.ScheduleCallback}
    ]
end
```

## Passo 3: Agente de Telemarketing com FSM

```elixir
defmodule MyApp.Telemarketing.Agent do
  use Jido.Agent,
    name: "telemarketing_agent",
    description: "Robô de telemarketing autônomo com fluxo FSM",
    strategy: {Jido.Agent.Strategy.FSM,
      initial_state: "idle",
      transitions: %{
        "idle"      => ["greeting"],
        "greeting"  => ["pitch", "done"],
        "pitch"     => ["objection", "closing", "done"],
        "objection" => ["pitch", "closing", "done"],
        "closing"   => ["done"],
        "done"      => ["idle"]
      }
    },
    plugins: [
      MyApp.Telemarketing.Plugin
    ],
    schema: [
      total_calls: [type: :integer, default: 0],
      successful_calls: [type: :integer, default: 0]
    ],
    signal_routes: [
      {"call.start", MyApp.Telemarketing.Actions.StartCall},
      {"call.pitch", MyApp.Telemarketing.Actions.DeliverPitch},
      {"call.response", MyApp.Telemarketing.Actions.ProcessResponse},
      {"call.close", MyApp.Telemarketing.Actions.CloseCall},
      {"call.schedule_callback", MyApp.Telemarketing.Actions.ScheduleCallback}
    ]
end
```

## Passo 4: Uso Puro (Testes)

```elixir
alias MyApp.Telemarketing.Agent, as: TeleAgent
alias MyApp.Telemarketing.Actions

# 1. Criar o agente
agent = TeleAgent.new()

# 2. Iniciar ligação
{agent, []} = TeleAgent.cmd(agent, {Actions.StartCall, %{
  customer_name: "Maria",
  customer_phone: "11999887766",
  product: "Plano Premium de Internet 500MB"
}})

IO.puts("=== Saudação ===")
agent.state.messages |> List.last() |> Map.get(:content) |> IO.puts()
# => "Bom dia, Maria! Aqui é da empresa XYZ..."

# 3. Fazer o pitch
{agent, []} = TeleAgent.cmd(agent, Actions.DeliverPitch)

IO.puts("\n=== Pitch ===")
agent.state.messages |> List.last() |> Map.get(:content) |> IO.puts()

# 4. Cliente responde com objeção
{agent, []} = TeleAgent.cmd(agent, {Actions.ProcessResponse, %{
  customer_message: "Hmm, não sei... está caro para mim."
}})

IO.puts("\n=== Resposta à objeção ===")
agent.state.messages |> List.last() |> Map.get(:content) |> IO.puts()
IO.puts("Tentativas: #{agent.state.attempts}")

# 5. Cliente demonstra interesse
{agent, []} = TeleAgent.cmd(agent, {Actions.ProcessResponse, %{
  customer_message: "Bom, me conte mais sobre o desconto."
}})

IO.puts("\n=== Cliente interessado ===")
IO.puts("Fase: #{agent.state.phase}")

# 6. Fechar a venda
{agent, directives} = TeleAgent.cmd(agent, {Actions.CloseCall, %{
  outcome: :sale_closed
}})

IO.puts("\n=== Fechamento ===")
agent.state.messages |> List.last() |> Map.get(:content) |> IO.puts()
IO.puts("Outcome: #{agent.state.outcome}")
IO.puts("Directives emitidas: #{length(directives)}")
```

## Passo 5: Uso em Produção com AgentServer

```elixir
# Configurar a instância Jido
defmodule MyApp.Jido do
  use Jido, otp_app: :my_app
end

# No application.ex
children = [MyApp.Jido]
Supervisor.start_link(children, strategy: :one_for_one)

# Iniciar o agente
{:ok, pid} = MyApp.Jido.start_agent(MyApp.Telemarketing.Agent, id: "telemarketer-1")

# Enviar signals
alias Jido.Signal

# Iniciar ligação via signal
signal = Signal.new!("call.start", %{
  customer_name: "João",
  customer_phone: "11988776655",
  product: "Seguro Residencial Premium"
}, source: "/crm")

{:ok, agent} = Jido.AgentServer.call(pid, signal)

# Processar resposta do cliente
signal = Signal.new!("call.response", %{
  customer_message: "Quanto custa por mês?"
}, source: "/telephony")

{:ok, agent} = Jido.AgentServer.call(pid, signal)

# Ver última resposta do bot
agent.state.messages |> List.last() |> Map.get(:content)
```

## Passo 6: Orquestrando Múltiplas Ligações

Para um call center com múltiplos agentes simultâneos:

```elixir
defmodule MyApp.Telemarketing.CallCenter do
  @moduledoc "Orquestra múltiplos agentes de telemarketing"

  alias MyApp.Telemarketing.Agent, as: TeleAgent
  alias Jido.Signal

  def start_campaign(customer_list, product) do
    Enum.map(customer_list, fn customer ->
      agent_id = "telemarketer-#{customer.phone}"

      # Inicia um agente por cliente
      {:ok, pid} = MyApp.Jido.start_agent(TeleAgent, id: agent_id)

      # Inicia a ligação
      signal = Signal.new!("call.start", %{
        customer_name: customer.name,
        customer_phone: customer.phone,
        product: product
      }, source: "/campaign")

      Jido.AgentServer.cast(pid, signal)

      {agent_id, pid}
    end)
  end

  def check_results do
    MyApp.Jido.list_agents()
    |> Enum.map(fn {id, pid} ->
      {:ok, server_state} = Jido.AgentServer.state(pid)
      agent = server_state.agent

      %{
        id: id,
        customer: agent.state[:customer_name],
        phase: agent.state[:phase],
        outcome: agent.state[:outcome],
        messages: length(agent.state[:messages] || [])
      }
    end)
  end
end

# Uso:
customers = [
  %{name: "Maria", phone: "11999001122"},
  %{name: "João", phone: "11988334455"},
  %{name: "Ana", phone: "11977556677"}
]

MyApp.Telemarketing.CallCenter.start_campaign(customers, "Internet Fibra 500MB")

# Depois de processar respostas...
MyApp.Telemarketing.CallCenter.check_results()
# => [
#   %{id: "telemarketer-11999001122", customer: "Maria", phase: :closing, outcome: nil, messages: 6},
#   %{id: "telemarketer-11988334455", customer: "João", phase: :done, outcome: :declined, messages: 8},
#   %{id: "telemarketer-11977556677", customer: "Ana", phase: :pitch, outcome: nil, messages: 4}
# ]
```

## Passo 7: Sensor para Integração com Telefonia

```elixir
defmodule MyApp.Telemarketing.TelephonySensor do
  @moduledoc "Recebe eventos do sistema de telefonia (ex: Twilio, Asterisk)"
  use Jido.Sensor,
    name: "telephony_sensor",
    description: "Captura eventos de telefonia e envia como signals",
    schema: Zoi.object(%{
      provider: Zoi.string() |> Zoi.default("twilio")
    })

  @impl true
  def init(config, _ctx) do
    {:ok, %{provider: config.provider}, []}
  end

  @impl true
  def handle_event({:speech_to_text, %{text: text, call_id: call_id}}, state) do
    signal = Jido.Signal.new!(
      "call.response",
      %{customer_message: text},
      source: "/telephony/#{state.provider}",
      subject: "/calls/#{call_id}"
    )
    {:ok, state, [{:emit, signal}]}
  end

  def handle_event({:call_ended, %{call_id: call_id}}, state) do
    signal = Jido.Signal.new!(
      "call.close",
      %{outcome: :call_ended},
      source: "/telephony/#{state.provider}",
      subject: "/calls/#{call_id}"
    )
    {:ok, state, [{:emit, signal}]}
  end
end
```

## Diagrama do Fluxo FSM

```
  ┌───────┐
  │ idle  │
  └───┬───┘
      │ call.start
      ▼
  ┌──────────┐
  │ greeting │──────────────┐
  └────┬─────┘              │ (cliente pede para parar)
       │ call.pitch         ▼
  ┌────▼─────┐         ┌───────┐
  │  pitch   │────────►│ done  │
  └────┬─────┘    ▲    └───┬───┘
       │          │        │
       │ objeção  │ 2+     │ (reinicia)
       ▼          │        ▼
  ┌──────────┐    │    ┌───────┐
  │ objection├────┘    │ idle  │
  └────┬─────┘         └───────┘
       │ interesse
       ▼
  ┌──────────┐
  │ closing  │──► done ──► idle
  └──────────┘
```

## Resumo

| Componente | Módulo | Responsabilidade |
|-----------|--------|-----------------|
| **StartCall** | Action | Inicia ligação com saudação |
| **DeliverPitch** | Action | Apresenta o produto |
| **ProcessResponse** | Action | Classifica intenção + responde via LLM |
| **CloseCall** | Action | Encerra e emite resultado |
| **ScheduleCallback** | Action | Agenda retorno |
| **Plugin** | Plugin | Empacota tudo com estado isolado |
| **Agent** | Agent + FSM | Máquina de estados do fluxo |
| **CallCenter** | Módulo | Orquestra múltiplos agentes |
| **TelephonySensor** | Sensor | Integra com sistema de telefonia |
