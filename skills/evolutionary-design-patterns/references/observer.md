# Observer / Event-Driven

Um emissor publica um evento e ouvintes reagem de forma desacoplada. O emissor não sabe quem escuta nem quantos são.

## Quando usar
- Uma ação principal (criar pedido, cadastrar usuário) dispara efeitos secundários: e-mail, analytics, auditoria, cache, webhook.
- Um serviço cresce chamando cada vez mais dependências "só mais uma" no final da função.
- Trabalho lento ou opcional não deve atrasar a resposta ao usuário.

## Quando NÃO usar
- Só existe um efeito, ele é parte essencial da operação e precisa ser atômico com ela. Chame direto (ou use transação/outbox).
- O fluxo exige resposta síncrona do "ouvinte". Isso é uma chamada, não um evento.

## Princípios
1. **Evento = fato no passado**, imutável: `OrderPlaced`, `PaymentConfirmed`. Não use comandos disfarçados (`SendEmail`).
2. O evento carrega o necessário para reagir (ids e dados-chave), não o objeto de domínio inteiro e mutável.
3. **Isolamento de falhas**: erro em um ouvinte é registrado e não afeta o emissor nem os outros ouvintes.
4. **Não bloqueie o fluxo principal** com trabalho lento: use `asyncio`, fila (SQS, RabbitMQ, Kafka, Redis) ou worker em background.
5. Ouvintes devem ser **idempotentes**: o mesmo evento pode chegar duas vezes.
6. Se perder o evento for inaceitável, use o padrão *outbox* (gravar o evento na mesma transação do dado) em vez de publicar em memória.

## Exemplo (Python, em processo)

```python
import logging
from collections import defaultdict
from collections.abc import Callable
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class OrderPlaced:
    order_id: str
    customer_id: str
    total_cents: int


class EventBus:
    def __init__(self) -> None:
        self._handlers: dict[type, list[Callable]] = defaultdict(list)

    def subscribe(self, event_type: type, handler: Callable) -> None:
        self._handlers[event_type].append(handler)

    def publish(self, event: object) -> None:
        for handler in self._handlers[type(event)]:
            try:
                handler(event)
            except Exception:
                # Falha de um ouvinte não derruba o fluxo principal nem os demais.
                logger.exception("Handler %s falhou para %s", handler.__name__, event)


# Emissor: conhece só o evento.
def place_order(bus: EventBus, order_id: str, customer_id: str, total_cents: int) -> None:
    # ... persistir o pedido ...
    bus.publish(OrderPlaced(order_id, customer_id, total_cents))


# Ouvintes: registrados na composição da aplicação.
def track_analytics(event: OrderPlaced) -> None: ...
def send_confirmation_email(event: OrderPlaced) -> None: ...


bus = EventBus()
bus.subscribe(OrderPlaced, track_analytics)
bus.subscribe(OrderPlaced, send_confirmation_email)
```

Para tornar assíncrono, substitua a chamada direta por `asyncio.create_task(...)` / `asyncio.gather(..., return_exceptions=True)` ou enfileire o evento para um worker.

## Exemplo (TypeScript)

```ts
type Handler<E> = (event: E) => void | Promise<void>;

class EventBus {
  private handlers = new Map<string, Handler<any>[]>();

  subscribe<E>(type: string, handler: Handler<E>) {
    this.handlers.set(type, [...(this.handlers.get(type) ?? []), handler]);
  }

  async publish<E>(type: string, event: E) {
    const results = await Promise.allSettled(
      (this.handlers.get(type) ?? []).map((h) => h(event)),
    );
    for (const r of results) {
      if (r.status === "rejected") console.error("Handler falhou", r.reason);
    }
  }
}
```

## Cuidados
- Muitos eventos em cadeia dificultam entender o fluxo. Documente os eventos e seus ouvintes principais.
- Evite ciclos (um ouvinte que publica o evento que o dispara).
- Teste o emissor com um bus falso que só registra os eventos publicados, e cada ouvinte separadamente.
