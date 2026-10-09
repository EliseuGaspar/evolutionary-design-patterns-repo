# evolutionary-design-patterns

Skill para agentes de código (Claude Code e compatíveis) que orienta a IA a escrever código fácil de mudar e de manter, aplicando cinco design patterns:

1. **Strategy**: troca cadeias de `if/else` por variantes intercambiáveis.
2. **Observer / event-driven**: efeitos colaterais desacoplados da ação principal.
3. **Adapter**: isola SDKs e APIs de terceiros atrás de interfaces do seu domínio.
4. **State Machine**: fluxos com estados e transições explícitas (inclui agentes de IA).
5. **Retry + Circuit Breaker**: resiliência contra falhas de serviços externos.

O skill também ensina a IA a **não exagerar**: com uma única variante, um `if` simples continua sendo a melhor opção.

## Instalação

### Via `npx skills` (recomendado)

```bash
npx skills add SEU-USUARIO/evolutionary-design-patterns
```

Para ver o que o repositório contém antes de instalar:

```bash
npx skills add SEU-USUARIO/evolutionary-design-patterns --list
```

### Manual com git

```bash
git clone https://github.com/SEU-USUARIO/evolutionary-design-patterns.git
mkdir -p ~/.claude/skills
cp -r evolutionary-design-patterns/skills/evolutionary-design-patterns ~/.claude/skills/
```

Para instalar só em um projeto, copie para `.claude/skills/` na raiz dele.

### Manual com curl (apenas o SKILL.md)

```bash
mkdir -p ~/.claude/skills/evolutionary-design-patterns
curl -o ~/.claude/skills/evolutionary-design-patterns/SKILL.md \
  https://raw.githubusercontent.com/SEU-USUARIO/evolutionary-design-patterns/main/skills/evolutionary-design-patterns/SKILL.md
```

> Atenção: o `curl` baixa só o `SKILL.md`. Os arquivos em `references/` (exemplos de cada padrão) não vêm junto. Para a versão completa, use `npx skills` ou `git clone`.

Depois de instalar, abra uma nova sessão do Claude Code. O skill é carregado automaticamente quando a tarefa envolve código de aplicação.

## Estrutura

```
skills/
└── evolutionary-design-patterns/
    ├── SKILL.md
    └── references/
        ├── strategy.md
        ├── observer.md
        ├── adapter.md
        ├── state-machine.md
        └── resilience.md
```

## Licença

MIT. Veja [LICENSE](LICENSE).
