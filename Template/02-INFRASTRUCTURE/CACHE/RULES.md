# RULES — Infrastructure / Cache

> Herda todas as regras de [`00-PRINCIPLES/ARCHITECTURE-RULES.md`](../../00-PRINCIPLES/ARCHITECTURE-RULES.md). Este documento especializa, nunca substitui.

## 1. Missão

Esta camada fornece **cache técnico genérico** (Redis) para qualquer módulo
usar. Assim como Database e Messaging, ela não sabe o que é um "Pedido" — só
guarda e devolve valores por chave, com expiração.

## 2. Decisões técnicas desta camada

| Decisão | Escolha |
|---|---|
| Tecnologia | **Redis** |
| Serialização | **JSON** via `System.Text.Json` (sem dependência extra) |
| Padrão de acesso | **Cache-aside** (leitura tenta cache, na ausência busca a fonte e popula) |
| TTL | **Sempre obrigatório** — não existe entrada de cache sem expiração |
| Isolamento entre módulos | Instância única de Redis, **namespace de chave por módulo** (mesmo princípio do schema por módulo no banco — ver `DATABASE/RULES.md`) |

## 3. Estrutura de pastas

`Cache` é uma subpasta dentro do projeto único `Infrastructure`
(`Infrastructure/Infrastructure.csproj`), assim como `Database` e `Messaging`
(`DATABASE/RULES.md` seção 3) — namespace `Infrastructure.Cache`.

```
Infrastructure/
└── Cache/
    ├── Interface/
    │   └── ICacheService.cs        # contrato usado pelos módulos
    ├── Implementation/
    │   └── RedisCacheService.cs    # única implementação concreta
    └── Extensions/
        └── CacheExtensions.cs      # AddCache(IServiceCollection, IConfiguration)
```

Subpasta por tipo de arquivo (`Interface/`, `Implementation/`, `Extensions/`)
pelo mesmo motivo de `Database` (`DATABASE/RULES.md` seção 3.1) — legibilidade
para quem nunca viu a pasta. `Cache` não tem `Factory/` nem `Map/` porque não
existe fábrica nem type map aqui; não criar pasta vazia só para manter
simetria com `Database`.

## 4. `ICacheService` — contrato

```csharp
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken ct = default);
    Task SetAsync<T>(string key, T value, TimeSpan expiration, CancellationToken ct = default);
    Task RemoveAsync(string key, CancellationToken ct = default);
    Task<T> GetOrSetAsync<T>(string key, Func<Task<T>> factory, TimeSpan expiration, CancellationToken ct = default);
}
```

- **Não existe overload de `SetAsync`/`GetOrSetAsync` sem `TimeSpan expiration`.** Cache sem TTL é a causa mais comum de dado obsoleto invisível — a API força a decisão de expiração a ser explícita em todo uso, sem exceção.
- Módulo nunca instancia `IConnectionMultiplexer`/cliente Redis diretamente — sempre via `ICacheService` injetado.

## 5. Convenção de chaves — namespace por módulo

Toda chave é prefixada pelo nome do schema do módulo dono do dado, seguindo o
mesmo padrão já usado no banco:

```
<schema-do-modulo>:<entidade>:<identificador>[:<variação>]
```

Exemplos: `vendas:pedido:123`, `estoque:produto:456:disponibilidade`.

- Um módulo nunca lê ou escreve uma chave fora do seu próprio prefixo. Se o Módulo B precisa de um dado que é "dono" do Módulo A, ele chama a interface pública (`Contracts`) do Módulo A — que internamente pode usar seu próprio cache — nunca lê a chave `vendas:*` diretamente.
- Essa regra espelha o isolamento de schema de banco (`DATABASE/RULES.md` seção 6): cache é dado derivado, mas a fronteira de propriedade é a mesma.

## 6. Padrão de uso — cache-aside

Cache nunca é fonte de verdade — é sempre um atalho para não recalcular/reconsultar algo que já está no banco (fonte de verdade real).

```csharp
var pedido = await _cache.GetOrSetAsync(
    key: $"vendas:pedido:{id}",
    factory: () => _pedidoRepository.ObterPorIdAsync(id),
    expiration: TimeSpan.FromMinutes(5));
```

- `GetOrSetAsync` é o padrão de leitura recomendado — módulo raramente deveria chamar `GetAsync`/`SetAsync` separados manualmente, a menos que o fluxo de fallback não seja um simples "buscar e cachear" (ex: quando o valor de fallback vem de um cálculo caro, não de uma única chamada de Repository).
- TTL é decisão do módulo, baseada em quão tolerável é o dado ficar desatualizado — não existe um valor padrão global; cada `GetOrSetAsync` declara o seu.

## 7. Invalidação

- Toda `Command`/operação que altera um dado que está (ou pode estar) cacheado é responsável por invalidar (`RemoveAsync`) a(s) chave(s) afetada(s) **no mesmo Handler**, antes de retornar sucesso. TTL é rede de segurança, não estratégia principal de consistência para dado sensível a ficar desatualizado.
- Se o dado é amplamente tolerante a desatualização (ex: um contador aproximado), pode-se confiar só no TTL curto sem invalidação explícita — decisão documentada no código (comentário curto ou nome do método) quando essa escolha for deliberada.

## 8. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Cache usado como única fonte de um dado (sem equivalente persistido no banco) | Cache pode ser evictado a qualquer momento — nunca é armazenamento durável |
| Entrada de cache sem TTL | Dado obsoleto que nunca expira sozinho |
| Módulo lendo/escrevendo chave fora do seu próprio prefixo `<schema>:` | Viola isolamento de módulo |
| Cliente Redis (`IConnectionMultiplexer`) usado diretamente fora de `Infrastructure/Cache` | Contorna a abstração, impede trocar implementação/instrumentar tudo num só lugar |
| Serializar tipo com referência circular ou `Entity` de domínio completa (em vez de DTO) no valor cacheado | Custo de serialização desnecessário e acopla o cache ao modelo interno do módulo |

## 9. Enforcement

- Code review bloqueia uso de `IConnectionMultiplexer` fora de `Infrastructure/Cache` e qualquer `SetAsync` sem TTL explícito (a própria assinatura da interface já impede isso em tempo de compilação).
- Code review bloqueia leitura/escrita de chave fora do prefixo `<schema-do-proprio-modulo>:`.
