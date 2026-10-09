---
name: evolutionary-design-patterns
description: Guia a IA a escrever código fácil de mudar e de manter, aplicando cinco design patterns - Strategy, Observer/event-driven, Adapter, State Machine e Retry + Circuit Breaker. Use SEMPRE que for escrever, refatorar ou revisar código de aplicação (backend, APIs, serviços, integrações, agentes de IA, workflows), mesmo que o usuário não peça "design patterns". Dispare especialmente quando houver cadeias de if/else ou switch por tipo, integração com SDK/API de terceiros (pagamento, e-mail, LLM, storage), efeitos colaterais após uma ação (notificar, logar, analytics), fluxos com estados e transições (pedido, aprovação, onboarding, agente de IA), ou chamadas a serviços externos que podem falhar (timeout, rate limit, instabilidade). Also use when writing or reviewing Python/TypeScript/Java/Go code that calls external services, branches on type, or models a lifecycle.
---

# Design Patterns para Evolução de Software

Software muda o tempo todo: novos requisitos, novos fornecedores, novos ouvintes de um evento, novos estados, serviços externos que caem. Código que não antecipa isso fica caro de alterar, e cada alteração arrisca quebrar o que já funcionava.

Estes cinco padrões existem para que **a mudança mais provável seja barata**: adicionar em vez de editar, trocar em vez de reescrever, falhar de forma controlada em vez de em cascata. Todos servem a um mesmo objetivo, o **desacoplamento**: cada parte conhece o mínimo possível das outras.

## Como trabalhar

1. **Antes de codar, pergunte: o que provavelmente vai mudar?** Novas variantes de um comportamento? Troca de fornecedor? Novos reagentes a um evento? Novos estados? Falha de algo externo?
2. **Procure o sinal na tabela abaixo** e leia o arquivo de referência do padrão correspondente para ver o formato esperado e os exemplos.
3. **Aplique o mínimo necessário.** Um padrão deve remover dor real ou prevista com boa probabilidade, nunca ser decoração (veja "Equilíbrio").
4. **Diga ao usuário, em uma ou duas frases, qual padrão usou e por quê**, para que a decisão seja revisável. Exemplo: "Usei Strategy para os cálculos de frete: novas transportadoras entram como classes novas, sem tocar no `if/else`."

## Tabela de sinais

| Sinal no requisito ou no código | Padrão | Leia |
|---|---|---|
| `if/elif/else` ou `switch` que escolhe *como* fazer algo conforme tipo/modo/plano/país | **Strategy** | `references/strategy.md` |
| Uma ação precisa disparar efeitos colaterais (notificar, auditar, analytics, e-mail) que não são o objetivo principal dela | **Observer / eventos** | `references/observer.md` |
| Usa SDK ou API de terceiros (pagamento, e-mail, LLM, storage, SMS) dentro da lógica de negócio | **Adapter** | `references/adapter.md` |
| Entidade ou fluxo com ciclo de vida: flags booleanas combinadas, `status` com regras de "pode ir de X para Y", orquestração de agente de IA | **State Machine** | `references/state-machine.md` |
| Chamada de rede, banco remoto, fila ou LLM que pode dar timeout, 429, 5xx | **Retry + Circuit Breaker** | `references/resilience.md` |

Um mesmo código costuma pedir vários padrões juntos. Por exemplo, um pagamento usa Adapter (isola o SDK), Retry + Circuit Breaker (protege a chamada) e Observer (emite `PaymentConfirmed` para quem precisar reagir).

## Resumo operacional de cada padrão

**Strategy.** Defina uma interface pequena (`Protocol`, `interface`, classe abstrata) e uma implementação por variante. O código cliente recebe a estratégia por injeção ou por um registro (dicionário/mapa), nunca decide por `if`. Adicionar variante = adicionar classe e registrar. Ganha testabilidade: cada estratégia é testada isoladamente.

**Observer / event-driven.** Quem origina o evento só publica um fato no passado (`OrderPlaced`, `UserSignedUp`) e não conhece os ouvintes. Ouvintes se inscrevem sozinhos. Uma falha em um ouvinte não pode derrubar o fluxo principal nem os outros ouvintes. Trabalho lento (analytics, e-mail) deve rodar de forma assíncrona ou em fila, para não bloquear a resposta ao usuário.

**Adapter.** Defina uma interface **do seu domínio** (`PaymentGateway.charge(...)`), nunca a do fornecedor. Uma classe adaptadora traduz entre ela e o SDK real, incluindo conversão de erros e tipos. O resto da aplicação importa só a interface, então trocar de fornecedor significa escrever outro adaptador. Tipos do SDK não devem vazar para fora do adaptador.

**State Machine.** Declare estados e transições permitidas como **dados** (um mapa `(estado, evento) -> novo_estado`), não como `if`s espalhados. Transição não prevista lança erro explícito. Ações de entrada/saída ficam em um único lugar. Para agentes de IA, modele fases (planejar, executar ferramenta, revisar, concluir, falhar) com limite de passos para evitar laços infinitos.

**Retry + Circuit Breaker.** Retry apenas para falhas **transitórias** (timeout, 429, 503), com backoff exponencial **e jitter**, teto de tentativas e teto de espera. Nunca repita erros de cliente (400, 401, 404) e só repita operações idempotentes (ou use chave de idempotência). O Circuit Breaker abre após N falhas seguidas, rejeita chamadas imediatamente enquanto está aberto e testa a recuperação de forma gradual (half-open). Ordem usual: o retry envolve o breaker, e `CircuitOpenError` não é repetido.

## Equilíbrio: quando NÃO aplicar

Padrões mal aplicados criam complexidade sem retorno. Use julgamento:

- **Uma variante só, sem sinal de que haverá outra**: um `if` simples é melhor que Strategy. Ao aparecer a segunda ou terceira variante, ou se o requisito já indicar extensão (por exemplo, "suportar vários provedores"), refatore.
- **Scripts descartáveis, protótipos, notebooks**: priorize clareza e velocidade. Mencione o padrão como próximo passo, se fizer sentido.
- **Interface com uma única implementação e sem fronteira real**: não crie. A exceção é **fronteira com sistema externo**, onde o Adapter quase sempre compensa.
- **Dois ou três estados triviais**: um `enum` com validação simples pode bastar. Use mapa de transições quando houver regras de quem pode ir para onde.
- **Retry em tudo**: não. Em operação não idempotente ou com erro permanente, retry só piora o problema.
- **Convenções do projeto vencem**: se a base já usa um event bus, um padrão de injeção ou uma biblioteca de resiliência (por exemplo `tenacity`, `resilience4j`, `polly`, `opossum`), use o que existe em vez de criar do zero.

## Checklist antes de entregar

- [ ] Adicionar uma nova variante, ouvinte, fornecedor ou estado exige **só código novo**, sem editar lógica existente?
- [ ] O domínio importa apenas **interfaces próprias**, nunca tipos de SDK de terceiros?
- [ ] Falha de um ouvinte ou de um serviço externo é **contida** (log, métrica, erro tipado) em vez de se propagar em cascata?
- [ ] Transições de estado inválidas **falham de forma explícita**?
- [ ] Todo retry tem **teto, backoff com jitter** e só atua em erros transitórios?
- [ ] Há testes (ou pontos de teste fáceis) para cada estratégia, adaptador (com fake) e tabela de transições?
- [ ] O padrão usado se justifica pela mudança esperada, ou há **exagero**?

## Contexto de IA e agentes

Sistemas com LLM concentram quase todos estes problemas: chamar um modelo é chamar um **serviço externo instável** (Adapter + Retry + Circuit Breaker), o fluxo de um agente é uma **máquina de estados** (State Machine), a escolha de modelo/prompt/ferramenta por contexto é **Strategy**, e registrar uso, custo e auditoria de cada passo é um caso natural de **eventos** (Observer). Ao escrever código de agentes, comece pensando nesses quatro encaixes.
