# State Machine (Máquina de Estados)

Gerencia fluxos baseados em **estados definidos** e **transições permitidas**, em vez de condicionais confusas. Útil para pedidos, aprovações, onboarding, pagamentos e, em especial, para orquestrar agentes de IA.

## Quando usar
- Entidade com ciclo de vida e regras de "só pode ir de X para Y".
- Várias flags booleanas combinadas (`is_paid`, `is_shipped`, `is_cancelled`) que permitem combinações impossíveis.
- `if status == ...` repetido em vários arquivos.
- Fluxo de agente de IA com fases, ferramentas e critérios de parada.

## Quando NÃO usar
- Dois estados triviais sem regras (ativo/inativo). Um `enum` basta.
- Fluxos puramente lineares sem desvios nem falhas.

## Princípios
1. Estados são um `enum` fechado. Eventos também.
2. As transições são **dados**: um mapa `(estado, evento) -> novo_estado`. Ler o mapa é entender o fluxo inteiro.
3. Transição inexistente lança **erro explícito** (`InvalidTransition`), nunca é ignorada em silêncio.
4. Efeitos (cobrar, notificar, chamar ferramenta) ficam em **um ponto** ligado à transição, em vez de espalhados.
5. Estados terminais (`DONE`, `FAILED`, `CANCELLED`) não têm saída.
6. Persista o estado atual. Se for útil, persista também o histórico de transições (auditoria).
7. Combine com Observer: publique um evento a cada transição relevante.

## Exemplo (Python): pedido

```python
from enum import Enum, auto


class OrderState(Enum):
    DRAFT = auto()
    PENDING_PAYMENT = auto()
    PAID = auto()
    SHIPPED = auto()
    CANCELLED = auto()


class OrderEvent(Enum):
    SUBMIT = auto()
    PAY = auto()
    SHIP = auto()
    CANCEL = auto()


TRANSITIONS: dict[tuple[OrderState, OrderEvent], OrderState] = {
    (OrderState.DRAFT, OrderEvent.SUBMIT): OrderState.PENDING_PAYMENT,
    (OrderState.PENDING_PAYMENT, OrderEvent.PAY): OrderState.PAID,
    (OrderState.PENDING_PAYMENT, OrderEvent.CANCEL): OrderState.CANCELLED,
    (OrderState.PAID, OrderEvent.SHIP): OrderState.SHIPPED,
    (OrderState.PAID, OrderEvent.CANCEL): OrderState.CANCELLED,
}


class InvalidTransition(Exception):
    pass


def next_state(state: OrderState, event: OrderEvent) -> OrderState:
    try:
        return TRANSITIONS[(state, event)]
    except KeyError:
        raise InvalidTransition(f"{event.name} não é permitido em {state.name}") from None
```

## Exemplo (Python): fluxo de agente de IA

```python
from enum import Enum, auto


class AgentState(Enum):
    PLANNING = auto()
    CALLING_TOOL = auto()
    REVIEWING = auto()
    DONE = auto()
    FAILED = auto()


class AgentEvent(Enum):
    PLAN_READY = auto()
    TOOL_OK = auto()
    TOOL_ERROR = auto()
    NEEDS_MORE_WORK = auto()
    ACCEPTED = auto()
    LIMIT_REACHED = auto()


AGENT_TRANSITIONS = {
    (AgentState.PLANNING, AgentEvent.PLAN_READY): AgentState.CALLING_TOOL,
    (AgentState.CALLING_TOOL, AgentEvent.TOOL_OK): AgentState.REVIEWING,
    (AgentState.CALLING_TOOL, AgentEvent.TOOL_ERROR): AgentState.PLANNING,  # replanejar
    (AgentState.REVIEWING, AgentEvent.NEEDS_MORE_WORK): AgentState.PLANNING,
    (AgentState.REVIEWING, AgentEvent.ACCEPTED): AgentState.DONE,
    # Proteção contra laço infinito, válida em qualquer fase não terminal:
    (AgentState.PLANNING, AgentEvent.LIMIT_REACHED): AgentState.FAILED,
    (AgentState.CALLING_TOOL, AgentEvent.LIMIT_REACHED): AgentState.FAILED,
    (AgentState.REVIEWING, AgentEvent.LIMIT_REACHED): AgentState.FAILED,
}

MAX_STEPS = 20  # o laço principal conta passos e emite LIMIT_REACHED ao exceder
```

O laço principal consulta o estado atual, executa a ação daquela fase (chamar o modelo, rodar a ferramenta, avaliar o resultado), traduz o resultado em um evento e aplica a transição.

## Exemplo (TypeScript)

```ts
type State = "draft" | "pending_payment" | "paid" | "shipped" | "cancelled";
type Event = "submit" | "pay" | "ship" | "cancel";

const transitions: Record<string, State> = {
  "draft:submit": "pending_payment",
  "pending_payment:pay": "paid",
  "pending_payment:cancel": "cancelled",
  "paid:ship": "shipped",
  "paid:cancel": "cancelled",
};

export function nextState(state: State, event: Event): State {
  const next = transitions[`${state}:${event}`];
  if (!next) throw new Error(`Transição inválida: ${event} em ${state}`);
  return next;
}
```

## Cuidados
- Em concorrência, aplique a transição com **trava otimista ou transação** (por exemplo, `UPDATE ... WHERE status = :esperado`), para que duas requisições não apliquem a mesma transição.
- Teste a tabela inteira: cada transição válida funciona e as inválidas lançam erro.
- Para máquinas muito grandes, considere uma biblioteca (por exemplo `transitions` em Python ou XState em TypeScript) em vez de crescer a sua.
