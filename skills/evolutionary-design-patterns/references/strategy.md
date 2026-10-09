# Strategy

Famílias de algoritmos intercambiáveis atrás de uma mesma interface. Elimina cadeias de `if/else` e `switch` que escolhem *como* fazer algo.

## Quando usar
- Cálculo ou comportamento varia por tipo, plano, país, canal, formato ou modelo (frete, desconto, imposto, exportação, escolha de LLM).
- A cada requisito novo alguém "adiciona mais um `elif`".
- Cada ramo precisa ser testado separadamente.

## Quando NÃO usar
- Só existe uma variante e nada indica que haverá outra.
- A variação é um valor (um número, uma string), não um comportamento. Use configuração.

## Estrutura
1. Interface mínima (um método, se possível).
2. Uma implementação por variante, sem estado compartilhado escondido.
3. Um registro (dict/map) ou injeção de dependência que escolhe a estratégia **uma vez**, na borda.
4. O código cliente só chama `strategy.metodo(...)`.

## Antes (cheiro)

```python
def calculate_shipping(order, carrier):
    if carrier == "correios":
        return order.weight * 2.5
    elif carrier == "dhl":
        return order.weight * 4.0 + 10
    elif carrier == "pickup":
        return 0
    else:
        raise ValueError(carrier)
```

## Depois (Python)

```python
from decimal import Decimal
from typing import Protocol


class ShippingStrategy(Protocol):
    def cost(self, weight_kg: Decimal) -> Decimal: ...


class CorreiosShipping:
    def cost(self, weight_kg: Decimal) -> Decimal:
        return weight_kg * Decimal("2.5")


class DhlShipping:
    def cost(self, weight_kg: Decimal) -> Decimal:
        return weight_kg * Decimal("4.0") + Decimal("10")


class PickupShipping:
    def cost(self, weight_kg: Decimal) -> Decimal:
        return Decimal("0")


SHIPPING_STRATEGIES: dict[str, ShippingStrategy] = {
    "correios": CorreiosShipping(),
    "dhl": DhlShipping(),
    "pickup": PickupShipping(),
}


def calculate_shipping(weight_kg: Decimal, carrier: str) -> Decimal:
    try:
        strategy = SHIPPING_STRATEGIES[carrier]
    except KeyError:
        raise ValueError(f"Transportadora desconhecida: {carrier}") from None
    return strategy.cost(weight_kg)
```

## Depois (TypeScript)

```ts
interface ShippingStrategy {
  cost(weightKg: number): number;
}

const shippingStrategies: Record<string, ShippingStrategy> = {
  correios: { cost: (w) => w * 2.5 },
  dhl: { cost: (w) => w * 4.0 + 10 },
  pickup: { cost: () => 0 },
};

export function calculateShipping(weightKg: number, carrier: string): number {
  const strategy = shippingStrategies[carrier];
  if (!strategy) throw new Error(`Transportadora desconhecida: ${carrier}`);
  return strategy.cost(weightKg);
}
```

## Cuidados
- Para variantes simples, uma função em vez de uma classe já é uma estratégia válida (`dict[str, Callable]`).
- Falhe cedo e de forma clara quando a chave não existir.
- Teste cada estratégia isoladamente e teste o registro (toda chave esperada existe).
