# Adapter

Tradutor entre interfaces incompatíveis. Na prática: isola SDKs e APIs de terceiros atrás de uma interface **do seu domínio**, para trocar de fornecedor sem tocar no núcleo da aplicação.

## Quando usar
- Pagamento, e-mail, SMS, storage, mapas, autenticação, LLM, analytics: qualquer fornecedor externo que você possa querer trocar, simular em testes ou usar em paralelo.
- O SDK tem tipos, erros ou convenções que você não quer espalhar pelo código.
- Você precisa de fakes simples para testes.

## Quando NÃO usar
- Biblioteca utilitária estável e universal (por exemplo, `datetime`, `json`). Não vale encapsular.
- Script de uso único.

## Princípios
1. A interface descreve o que **a sua aplicação precisa**, em linguagem do seu domínio (`charge`, `refund`), e não o que o fornecedor oferece.
2. O adaptador converte **entrada, saída e erros**. Exceções do SDK viram erros do seu domínio.
3. Tipos do SDK **não vazam** além do adaptador.
4. O adaptador é montado na composição da aplicação (injeção de dependência), nunca instanciado dentro da regra de negócio.
5. Configuração e credenciais entram pelo construtor.
6. Resiliência (retry, circuit breaker) normalmente se aplica **dentro** ou **ao redor** do adaptador. Veja `resilience.md`.

## Exemplo (Python)

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class PaymentResult:
    ok: bool
    transaction_id: str | None = None
    failure_reason: str | None = None


class PaymentError(Exception):
    """Erro de infraestrutura de pagamento (rede, indisponibilidade)."""


class PaymentGateway(Protocol):
    def charge(self, amount_cents: int, currency: str, payment_token: str) -> PaymentResult: ...


class AcmePayAdapter:
    """Adapta o SDK hipotético 'acmepay' à interface do domínio."""

    def __init__(self, client) -> None:
        self._client = client  # cliente do SDK, injetado

    def charge(self, amount_cents: int, currency: str, payment_token: str) -> PaymentResult:
        try:
            response = self._client.create_charge(
                amount=amount_cents, currency=currency.lower(), source=payment_token
            )
        except self._client.CardDeclined as exc:
            return PaymentResult(ok=False, failure_reason=str(exc))
        except self._client.NetworkError as exc:
            raise PaymentError("Gateway indisponível") from exc
        return PaymentResult(ok=True, transaction_id=response.id)


class FakePaymentGateway:
    """Para testes: sem rede, sem SDK."""

    def charge(self, amount_cents: int, currency: str, payment_token: str) -> PaymentResult:
        return PaymentResult(ok=True, transaction_id="fake-123")


# Regra de negócio: só conhece a interface.
def checkout(gateway: PaymentGateway, amount_cents: int, token: str) -> PaymentResult:
    return gateway.charge(amount_cents, "BRL", token)
```

Para trocar de fornecedor, escreva `OtherPayAdapter` e altere uma linha na composição. `checkout` não muda.

## Exemplo (TypeScript)

```ts
export interface PaymentGateway {
  charge(amountCents: number, currency: string, token: string): Promise<PaymentResult>;
}

export type PaymentResult =
  | { ok: true; transactionId: string }
  | { ok: false; failureReason: string };

export class AcmePayAdapter implements PaymentGateway {
  constructor(private readonly client: AcmePayClient) {}

  async charge(amountCents: number, currency: string, token: string): Promise<PaymentResult> {
    try {
      const r = await this.client.createCharge({ amount: amountCents, currency, source: token });
      return { ok: true, transactionId: r.id };
    } catch (e) {
      if (e instanceof CardDeclinedError) return { ok: false, failureReason: e.message };
      throw new PaymentError("Gateway indisponível", { cause: e });
    }
  }
}
```

## Caso de LLM
Defina `LLMClient.complete(messages, **opts) -> Completion` com tipos seus (texto, uso de tokens, motivo de parada). Um adaptador por provedor traduz formato de mensagens, parâmetros e erros (rate limit, timeout). O restante do sistema não importa o SDK do provedor.

## Cuidados
- Evite interfaces que reproduzam 1:1 o SDK. Isso é só um proxy, e a troca de fornecedor continuaria dolorosa.
- Cubra o adaptador com testes de contrato (mesmos testes rodando contra o fake e, se possível, contra o sandbox real).
