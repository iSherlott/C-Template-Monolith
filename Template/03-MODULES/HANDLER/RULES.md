# RULES — Modules / Handler

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Handler` é onde a **regra de aplicação** vive: orquestra `Repository`,
`Service`, `IEventBus`, `ICacheService` e, quando necessário, a interface
pública de outro módulo, para executar um único `Command` ou `Query`. É o
coração do módulo — mas não conhece HTTP, não conhece SQL, e não conhece o
funcionamento interno de nenhum outro módulo.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Handler/
    └── <Verbo><Recurso>Handler.cs    # ex: CriarPedidoHandler.cs, ObterPedidoPorIdHandler.cs
```

Regra fixa: **um `Handler` por `Command`/`Query`** (`ARCHITECTURE-RULES.md`
seção 7). Nunca um `Handler` genérico tratando múltiplos comandos por
`switch`/`if` de tipo.

## 3. Assinatura padrão

```csharp
public class CriarPedidoHandler
{
    private readonly IPedidoRepository _pedidoRepository;
    private readonly IUnitOfWorkFactory _unitOfWorkFactory;
    private readonly IEventBus _eventBus;

    public CriarPedidoHandler(
        IPedidoRepository pedidoRepository,
        IUnitOfWorkFactory unitOfWorkFactory,
        IEventBus eventBus)
    {
        _pedidoRepository = pedidoRepository;
        _unitOfWorkFactory = unitOfWorkFactory;
        _eventBus = eventBus;
    }

    public async Task<Result<PedidoDto>> Handle(CriarPedidoCommand command)
    {
        if (command.Itens.Count == 0)
            return Result<PedidoDto>.Failure(Error.Validation("Pedido precisa ter ao menos um item."));

        var pedido = Pedido.Criar(command.ClienteId, command.Itens);

        using var unitOfWork = _unitOfWorkFactory.Create();

        await _pedidoRepository.InserirAsync(pedido, unitOfWork);
        await _eventBus.PublishAsync(new PedidoCriadoEvent(pedido.Id, pedido.ClienteId), "vendas", unitOfWork);

        unitOfWork.Commit();

        return Result<PedidoDto>.Success(PedidoDto.FromEntity(pedido));
    }
}
```

- Método público sempre chamado `Handle`, recebendo o `Command`/`Query` como único parâmetro.
- Retorno sempre `Task<Result<TDto>>` — nunca `Task<Entity>`, nunca `void`, nunca lança exceção para representar uma falha de negócio esperada (ver seção 6).
- Lifetime de registro no DI: `Scoped` (consistente com `03-MODULES/RULES.md` seção 6).
- **`Handler` nunca instancia `IDbConnection`/`IDbTransaction` diretamente** — a conexão é aberta e fechada por dentro de `IUnitOfWorkFactory.Create()` (`DATABASE/RULES.md` seção 5). O `Handler` só decide *quando* confirmar (`unitOfWork.Commit()`), nunca *como* a conexão é gerenciada.

## 4. O que um Handler orquestra

| Dependência | Quando usar |
|---|---|
| `Repository` (próprio módulo) | Ler/gravar dado que pertence a este módulo |
| `Service` (próprio módulo) | Reaproveitar lógica de domínio usada por mais de um Handler (ver `SERVICES/RULES.md`) |
| `IUnitOfWorkFactory` | Quando o fluxo grava em um ou mais `Repository` e precisa de um `IUnitOfWork` pra passar adiante (`DATABASE/RULES.md` seção 5) |
| `IEventBus` | Publicar um `IntegrationEvent` — sempre com o mesmo `IUnitOfWork` da escrita (`MESSAGING/RULES.md` seção 5) |
| `ICacheService` | Ler/invalidar cache do próprio módulo (`CACHE/RULES.md`) |
| `I<NomeModulo>` de outro módulo | Quando precisa de dado/ação síncrona que pertence a outro módulo (`ARCHITECTURE-RULES.md` seção 4) |

Um `Query`-Handler simples (sem escrita) normalmente só usa `Repository`
(leitura) e/ou `ICacheService` — não precisa de `IUnitOfWork`, já que não há
nada para tornar atômico.

Quando o fluxo grava em **mais de um** `Repository` (ou grava e publica
evento), o mesmo `IUnitOfWork` é passado para cada chamada — não existe
parâmetro booleano decidindo "commita agora ou espera"; um único `Commit()`
no final decide o destino de tudo que usou aquele `IUnitOfWork`:

```csharp
using var unitOfWork = _unitOfWorkFactory.Create();

await _pedidoRepository.InserirAsync(pedido, unitOfWork);
await _estoqueLocalRepository.DebitarAsync(item, unitOfWork); // mesmo unitOfWork — mesma transação
await _eventBus.PublishAsync(new PedidoCriadoEvent(pedido.Id), "vendas", unitOfWork);

unitOfWork.Commit(); // só aqui tudo acima é confirmado junto
```

Se qualquer chamada lançar exceção antes do `Commit()`, o `using` chama
`Dispose()` no `IUnitOfWork`, que reverte a transação inteira automaticamente
— não é necessário `try/catch` manual para o rollback.

## 5. Handler chamando outro módulo

Quando o fluxo depende de outro módulo, o `Handler` injeta a interface
pública desse módulo — nunca `Repository`/`Service` interno dele:

```csharp
public class CriarPedidoHandler
{
    private readonly IEstoqueModule _estoqueModule; // Shared.Contracts — ver CONTRACTS/RULES.md seção 1

    public async Task<Result<PedidoDto>> Handle(CriarPedidoCommand command)
    {
        var disponivel = await _estoqueModule.VerificarDisponibilidadeAsync(command.Itens);
        if (!disponivel)
            return Result<PedidoDto>.Failure(Error.Conflict("Item sem estoque disponível."));

        // ...
    }
}
```

Isso é a aplicação prática do canal síncrono definido em
`ARCHITECTURE-RULES.md` seção 4 — o `Handler` é onde esse canal é, de fato,
usado. `IEstoqueModule` vive em `Modules/Shared/Contracts/` (`CONTRACTS/RULES.md`
seção 1), não dentro do módulo Estoque — é por isso que injetar essa
interface nunca exige `ProjectReference` de um módulo de negócio para outro.

## 6. Falha esperada (`Result.Failure`) vs. exceção não tratada

- **Falha esperada** — uma condição de negócio prevista (validação, recurso não encontrado, conflito, não autorizado) sempre vira `Result<T>.Failure(Error)`, nunca uma exceção lançada. É assim que o `Controller` sabe traduzir para `400`/`404`/`409` (`CONTROLLER/RULES.md` seção 5).
- **Falha inesperada** — erro de infraestrutura (banco fora do ar, exceção de bug) **não é capturada e empacotada como `Result.Failure` dentro do Handler**. Ela simplesmente propaga (`throw`), e é responsabilidade do middleware global de exceção no Host (`01-HOST/RULES.md` seção 5) transformar isso em `500`.
- Regra prática: se o `Handler` está escrevendo um `try/catch` para converter uma exceção de infraestrutura em `Result.Failure(Error.Unexpected(...))`, questionar se isso não deveria simplesmente propagar. Captura manual só se houver uma ação de recuperação real (ex: um retry pontual), não só para "não deixar exceção subir".

## 7. Mapeamento Entity → Dto

`Handler` é o único lugar que conhece tanto a `Entity` (privada) quanto o
`Dto` (público, em `Contracts`). A conversão acontece aqui — nunca no
`Repository` (que deveria poder retornar `Entity` pura) nem no `Controller`
(que nunca deveria ver `Entity`). Convenção: método estático `Dto.FromEntity(entity)`
vivendo na própria classe do `Dto`.

## 8. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Handler` retornando `Entity` em vez de `Dto` | Vaza modelo interno para fora do módulo |
| `Handler` com SQL embutido | SQL pertence exclusivamente ao `Repository` |
| `Handler` injetando `Repository`/`Service` de outro módulo | Viola isolamento — deve usar `I<NomeModulo>` (`Contracts`) |
| `Handler` conhecendo `HttpContext`/`IActionResult` | Acopla regra de aplicação ao transporte HTTP |
| `Handler` instanciando `IDbConnection`/`IDbTransaction` diretamente | Gerenciamento de conexão é responsabilidade de `IUnitOfWorkFactory` (`DATABASE/RULES.md` seção 5), não do Handler |
| Parâmetro booleano tipo `commit: bool` num método de `Repository` pra controlar se finaliza a transação | O controle de commit/rollback é sempre do `IUnitOfWork` (`.Commit()`/`Dispose()` automático), nunca de uma flag por chamada |
| Capturar exceção de infraestrutura só para virar `Result.Failure(Unexpected)` sem ação de recuperação | Duplica o que o middleware global já faz, esconde erro real do log de exceção |
| Um `Handler` processando mais de um tipo de `Command`/`Query` | Quebra a regra "um Handler por Command/Query" |

## 9. Enforcement

- Code review confere que toda falha de negócio esperada retorna `Result.Failure` com o `ErrorType` correto, nunca exceção customizada para esse fim.
- Code review bloqueia qualquer `using` de namespace de outro módulo de negócio (`Entities`/`Repositories`/`Services` de outro módulo) dentro de um Handler — só `Shared.Contracts` é permitido, e fisicamente é o único que compila, já que não há `ProjectReference` para nenhum outro módulo (`CONTRACTS/RULES.md` seção 2).
- Code review confere que o construtor do `Handler` continua `public`, mesmo que suas dependências (`Repository` implementação, `Service`) sejam `internal` — só a interface injetada precisa ser visível (`ARCHITECTURE-RULES.md` seção 5.1).
