# Retry + Circuit Breaker

Técnicas de resiliência para lidar com falhas de serviços externos (APIs, banco remoto, filas, LLMs) sem derrubar a aplicação.

## Quando usar
- Qualquer chamada de rede ou a um serviço que você não controla.
- Provedores de LLM (429, 529/overloaded, timeout), gateways de pagamento, APIs de terceiros.

## Quando NÃO usar
- Chamadas locais em memória.
- Retry em operação **não idempotente** sem chave de idempotência (risco de cobrar duas vezes).
- Retry em erros permanentes (400, 401, 403, 404, validação). Repetir não resolve.

## Retry: regras

1. Repita só **falhas transitórias**: timeout, erro de conexão, 429, 502, 503, 504.
2. **Backoff exponencial**: a espera cresce a cada tentativa (0,5s, 1s, 2s, 4s...).
3. **Jitter**: aleatoriedade na espera, para que muitos clientes não tentem todos ao mesmo tempo e sobrecarreguem o serviço que está se recuperando (efeito *thundering herd*).
4. **Tetos**: número máximo de tentativas e espera máxima.
5. Respeite o cabeçalho `Retry-After` quando existir.
6. Registre cada tentativa (log/métrica) para tornar o problema visível.

```python
import random
import time
from collections.abc import Callable
from typing import TypeVar

T = TypeVar("T")


def retry(
    fn: Callable[[], T],
    *,
    attempts: int = 5,
    base_delay: float = 0.5,
    max_delay: float = 30.0,
    retry_on: tuple[type[Exception], ...] = (TimeoutError, ConnectionError),
    sleep: Callable[[float], None] = time.sleep,  # injetável para testes
) -> T:
    for attempt in range(attempts):
        try:
            return fn()
        except retry_on:
            if attempt == attempts - 1:
                raise
            # "Full jitter": espera aleatória entre 0 e o teto exponencial.
            ceiling = min(max_delay, base_delay * (2**attempt))
            sleep(random.uniform(0, ceiling))
    raise AssertionError("inalcançável")
```

## Circuit Breaker: regras

Três estados:
- **CLOSED**: chamadas passam normalmente; falhas são contadas.
- **OPEN**: após N falhas seguidas, rejeita chamadas **imediatamente** (sem tocar no serviço) por um tempo.
- **HALF_OPEN**: passado o tempo, deixa passar uma chamada de teste. Se der certo, volta a CLOSED. Se falhar, volta a OPEN.

Benefícios: não desperdiça threads, tempo e dinheiro com um serviço que já está fora, e evita falhas em cascata. Falhar rápido ainda permite devolver um *fallback* (cache, resposta degradada, fila para depois).

```python
import time
from collections.abc import Callable
from enum import Enum
from typing import TypeVar

T = TypeVar("T")


class CircuitOpenError(Exception):
    pass


class _State(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


class CircuitBreaker:
    """Versão simples e NÃO thread-safe. Em código concorrente, proteja com lock
    ou use uma biblioteca madura."""

    def __init__(
        self,
        failure_threshold: int = 5,
        reset_timeout: float = 30.0,
        clock: Callable[[], float] = time.monotonic,  # injetável para testes
    ) -> None:
        self._failure_threshold = failure_threshold
        self._reset_timeout = reset_timeout
        self._clock = clock
        self._state = _State.CLOSED
        self._failures = 0
        self._opened_at = 0.0

    def call(self, fn: Callable[[], T]) -> T:
        if self._state is _State.OPEN:
            if self._clock() - self._opened_at >= self._reset_timeout:
                self._state = _State.HALF_OPEN
            else:
                raise CircuitOpenError("Circuito aberto: serviço indisponível")
        try:
            result = fn()
        except Exception:
            self._on_failure()
            raise
        self._on_success()
        return result

    def _on_success(self) -> None:
        self._failures = 0
        self._state = _State.CLOSED

    def _on_failure(self) -> None:
        self._failures += 1
        if self._state is _State.HALF_OPEN or self._failures >= self._failure_threshold:
            self._state = _State.OPEN
            self._opened_at = self._clock()
```

## Combinando os dois

O retry envolve o breaker, e `CircuitOpenError` **não** é repetido (se o circuito está aberto, insistir não adianta):

```python
breaker = CircuitBreaker(failure_threshold=5, reset_timeout=30)

def call_provider():
    return retry(
        lambda: breaker.call(lambda: http_client.post(url, json=payload, timeout=10)),
        retry_on=(TimeoutError, ConnectionError),  # CircuitOpenError fica de fora
    )
```

Coloque isto dentro do **adaptador** do serviço externo (ver `adapter.md`), para que o restante da aplicação receba apenas erros do seu domínio.

## TypeScript (esboço)

```ts
export async function retry<T>(
  fn: () => Promise<T>,
  { attempts = 5, baseDelayMs = 500, maxDelayMs = 30_000, isRetryable = (_e: unknown) => true } = {},
): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn();
    } catch (e) {
      if (attempt >= attempts - 1 || !isRetryable(e)) throw e;
      const ceiling = Math.min(maxDelayMs, baseDelayMs * 2 ** attempt);
      await new Promise((r) => setTimeout(r, Math.random() * ceiling));
    }
  }
}
```

Para o breaker em Node, prefira uma biblioteca estabelecida (por exemplo `opossum`).

## Cuidados
- Defina **timeouts** em toda chamada externa. Retry e breaker não ajudam se a chamada ficar pendurada para sempre.
- Idempotência: ao repetir operações que mudam estado, envie uma chave de idempotência.
- Observabilidade: exponha o estado do breaker e a contagem de retries como métricas.
- Teste com relógio e `sleep` injetados, sem esperar tempo real.
