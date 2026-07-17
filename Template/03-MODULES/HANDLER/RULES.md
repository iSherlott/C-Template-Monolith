# RULES — Modules / Handler

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Handler` é onde a **regra de aplicação** vive: orquestra `Repository`,
`Service`, `IEventBus`, `ICacheService` e, quando necessário, a interface
pública de outro módulo, para executar um `Command` (mutação ou leitura —
`COMMANDS/RULES.md` seção 4). É o coração do módulo — mas não conhece HTTP,
não conhece SQL, e não conhece o funcionamento interno de nenhum outro
módulo.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Handler/
    └── <Recurso>Handler.cs    # ex: PedidoHandler.cs — cobre TODOS os Commands do recurso "Pedido"
```

**Decisão:** um `Handler` por **recurso** (tipicamente, um por `Controller`
— seção 4 detalha a exceção de subpath), não um `Handler` por `Command`
individual. `PedidoHandler` implementa `IHandler<TCommand, TResult>`
(`SHARED/RULES.md`) **uma vez para cada** `Command` que o recurso aceita —
`CreatePedidoCommand`,
`CancelPedidoCommand`, `GetPedidoByIdCommand`, todos na mesma classe,
cada um com seu próprio método `Handle` (sobrecarregado por tipo de
parâmetro).

## 3. `IHandler<TCommand, TResult>` — o contrato, implementado várias vezes

```csharp
// Shared.Kernel — SHARED/RULES.md seção 3
public interface IHandler<in TCommand, TResult>
{
    Task<Result<TResult>> Handle(TCommand command);
}
```

```csharp
public class PedidoHandler :
    IHandler<CreatePedidoCommand, PedidoDto>,
    IHandler<GetPedidoByIdCommand, PedidoDto>
{
    private readonly IPedidoRepository _pedidoRepository;
    private readonly IUnitOfWorkFactory _unitOfWorkFactory;
    private readonly IEventBus _eventBus;

    public PedidoHandler(
        IPedidoRepository pedidoRepository,
        IUnitOfWorkFactory unitOfWorkFactory,
        IEventBus eventBus)
    {
        _pedidoRepository = pedidoRepository;
        _unitOfWorkFactory = unitOfWorkFactory;
        _eventBus = eventBus;
    }

    public async Task<Result<PedidoDto>> Handle(CreatePedidoCommand command)
    {
        if (command.Itens.Count == 0)
            return Result<PedidoDto>.Failure(Error.Validation(PedidoDictionary.PedidoSemItens));

        var pedido = Pedido.Criar(command.ClienteId, command.Itens);

        using var unitOfWork = _unitOfWorkFactory.Create();

        await _pedidoRepository.InserirAsync(pedido, unitOfWork);
        await _eventBus.PublishAsync(new PedidoCriadoEvent(pedido.Id, pedido.ClienteId), "vendas", unitOfWork);

        unitOfWork.Commit();

        return Result<PedidoDto>.Success(PedidoDto.FromEntity(pedido));
    }

    public async Task<Result<PedidoDto>> Handle(GetPedidoByIdCommand command)
    {
        var pedido = await _pedidoRepository.ObterPorIdAsync(command.PedidoId);
        if (pedido is null)
            return Result<PedidoDto>.Failure(Error.NotFound(PedidoDictionary.PedidoNaoEncontrado));

        return Result<PedidoDto>.Success(PedidoDto.FromEntity(pedido));
    }
}
```

- Cada método `Handle` é público, aceita **um** `Command` como único
  parâmetro, e o overload correto é resolvido pelo compilador no
  `Controller` (mesmo tipo estático do argumento — não precisa de reflection
  nem de `switch` de tipo dentro do `Handler`). Isso preserva a decisão "sem
  mediator" (`ARCHITECTURE-RULES.md` seção 7) — o `Controller` continua
  chamando o método diretamente, só que agora numa classe que cobre mais de
  um `Command`.
- Retorno sempre `Task<Result<TDto>>` — nunca `Task<Entity>`, nunca `void`, nunca lança exceção para representar uma falha de negócio esperada (ver seção 7).
- Lifetime de registro no DI: `Scoped` (consistente com `03-MODULES/RULES.md` seção 6). Uma única instância cobre todos os `Command` do recurso na mesma requisição.
- **`Handler` nunca instancia `IDbConnection`/`IDbTransaction` diretamente** — a conexão é aberta e fechada por dentro de `IUnitOfWorkFactory.Create()` (`DATABASE/RULES.md` seção 5). O `Handler` só decide *quando* confirmar (`unitOfWork.Commit()`), nunca *como* a conexão é gerenciada.
- Dependências que só um dos `Handle` usa (ex: `IEventBus`, usado só por `CreatePedidoCommand`) continuam injetadas no construtor único da classe — não há problema em uma dependência ficar "parada" durante o `Handle` de outro `Command`, o custo de injeção é desprezível.

## 4. Quando separar em `Handler`s diferentes — recurso distinto no mesmo Controller

Se um único `Controller` expõe **subcaminhos que são recursos
substantivamente diferentes** (não variações de verbo HTTP sobre o mesmo
substantivo), cada subcaminho ganha seu próprio `Handler`. Exemplo real:
`CatalogoController` responde tanto por `api/catalogo/generos` quanto por
`api/catalogo/animes` — "gênero" e "anime" são substantivos diferentes, cada
um com seu próprio ciclo de vida e Aggregate Root — então o módulo tem
`GeneroHandler` **e** `AnimeHandler`, e o `Controller` injeta os dois:

```csharp
[ApiController]
[Route("api/catalogo")]
public class CatalogoController : ControllerBase
{
    private readonly GeneroHandler _generoHandler;
    private readonly AnimeHandler _animeHandler;

    public CatalogoController(GeneroHandler generoHandler, AnimeHandler animeHandler)
    {
        _generoHandler = generoHandler;
        _animeHandler = animeHandler;
    }

    [HttpPost("generos")]
    public async Task<IActionResult> PostGenero([FromBody] CreateGeneroRequest request) =>
        (await _generoHandler.Handle(new CreateGeneroCommand(request.Nome))).ToActionResult(StatusCodes.Status201Created);

    [HttpPost("animes")]
    public async Task<IActionResult> PostAnime([FromBody] CreateAnimeCommand command) =>
        (await _animeHandler.Handle(command)).ToActionResult(StatusCodes.Status201Created);
}
```

**Regra de decisão — "banana vs. tomate":** se dois subcaminhos do mesmo
`Controller` nomeiam **substantivos diferentes** (`generos` vs. `animes`,
"banana" vs. "tomate"), são `Handler`s separados. Se os subcaminhos são
**ações diferentes sobre o mesmo substantivo** (`login`/`registrar` sob
`auth`, `progresso` sob `watchlist`), continuam no mesmo `Handler` — não há
um segundo substantivo para justificar a separação. Na dúvida, o teste
prático é: "esses dois subcaminhos têm Aggregate Root diferente?" Se sim,
`Handler`s diferentes; se é a mesma Aggregate Root com operações diferentes,
um único `Handler`.

## 5. O que um Handler orquestra

| Dependência | Quando usar |
|---|---|
| `Repository` (próprio módulo) | Ler/gravar dado que pertence a este módulo |
| `Service` (próprio módulo) | Reaproveitar lógica de domínio usada por mais de um `Handle` (ver `SERVICES/RULES.md`) |
| `IUnitOfWorkFactory` | Quando o fluxo grava em um ou mais `Repository` e precisa de um `IUnitOfWork` pra passar adiante (`DATABASE/RULES.md` seção 5) |
| `IEventBus` | Publicar um `IntegrationEvent` — sempre com o mesmo `IUnitOfWork` da escrita (`MESSAGING/RULES.md` seção 5) |
| `ICacheService` | Ler/invalidar cache do próprio módulo (`CACHE/RULES.md`) |
| `I<NomeModulo>` de outro módulo | Quando precisa de dado/ação síncrona que pertence a outro módulo (`ARCHITECTURE-RULES.md` seção 4) |

Um `Handle` de leitura simples (`Get*Command`, sem escrita) normalmente só
usa `Repository` (leitura) e/ou `ICacheService` — não precisa de
`IUnitOfWork`, já que não há nada para tornar atômico.

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

## 6. Handler chamando outro módulo

Quando o fluxo depende de outro módulo, o `Handler` injeta a interface
pública desse módulo — nunca `Repository`/`Service` interno dele:

```csharp
public class PedidoHandler : IHandler<CreatePedidoCommand, PedidoDto>
{
    private readonly IEstoqueModule _estoqueModule; // Shared.Contracts — ver CONTRACTS/RULES.md seção 1

    public async Task<Result<PedidoDto>> Handle(CreatePedidoCommand command)
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

## 7. Falha esperada (`Result.Failure`) vs. exceção não tratada

- **Falha esperada** — uma condição de negócio prevista (validação, recurso não encontrado, conflito, não autorizado) sempre vira `Result<T>.Failure(Error)`, nunca uma exceção lançada. É assim que o `Controller` sabe traduzir para `400`/`404`/`409` (`CONTROLLER/RULES.md` seção 5).
- **Falha inesperada** — erro de infraestrutura (banco fora do ar, exceção de bug) **não é capturada e empacotada como `Result.Failure` dentro do Handler**. Ela simplesmente propaga (`throw`), e é responsabilidade do `GlobalExceptionMiddleware` (`02-INFRASTRUCTURE/SECURITY/RULES.md` seção 6) transformar isso em `500`.
- Regra prática: se o `Handler` está escrevendo um `try/catch` para converter uma exceção de infraestrutura em `Result.Failure(Error.Unexpected(...))`, questionar se isso não deveria simplesmente propagar. Captura manual só se houver uma ação de recuperação real (ex: um retry pontual), não só para "não deixar exceção subir".

**❌ Errado:**

```csharp
public async Task<Result<PedidoDto>> Handle(GetPedidoByIdCommand command)
{
    try
    {
        var pedido = await _pedidoRepository.ObterPorIdAsync(command.PedidoId);
        if (pedido is null)
            throw new Exception("Pedido não encontrado"); // ❌ falha esperada não deveria ser exceção
        return Result<PedidoDto>.Success(PedidoDto.FromEntity(pedido));
    }
    catch (Exception ex) // ❌ engole erro de infra (ex: timeout de conexão) e some com o stack trace real
    {
        return Result<PedidoDto>.Failure(Error.Unexpected(ex.Message));
    }
}
```

**✅ Correto:**

```csharp
public async Task<Result<PedidoDto>> Handle(GetPedidoByIdCommand command)
{
    var pedido = await _pedidoRepository.ObterPorIdAsync(command.PedidoId);
    if (pedido is null)
        return Result<PedidoDto>.Failure(Error.NotFound(PedidoDictionary.PedidoNaoEncontrado));

    return Result<PedidoDto>.Success(PedidoDto.FromEntity(pedido));
    // erro de infra (conexão caiu, etc.) propaga sozinho — GlobalExceptionMiddleware vira 500
}
```

## 8. Mapeamento Entity → Dto

`Handler` é o único lugar que conhece tanto a `Entity` (privada) quanto o
`Dto` (público, em `Contracts`). A conversão acontece aqui — nunca no
`Repository` (que deveria poder retornar `Entity` pura) nem no `Controller`
(que nunca deveria ver `Entity`). Convenção: método estático `Dto.FromEntity(entity)`
vivendo na própria classe do `Dto`.

**❌ Errado — Handler devolvendo a Entity direto:**

```csharp
public interface IHandler<in TCommand, TResult> { /* ... */ }

public class PedidoHandler : IHandler<GetPedidoByIdCommand, Pedido> // ❌ TResult é a Entity
{
    public async Task<Result<Pedido>> Handle(GetPedidoByIdCommand command) =>
        Result<Pedido>.Success(await _pedidoRepository.ObterPorIdAsync(command.PedidoId)); // ❌ vaza modelo interno
}
```

**✅ Correto — Handler sempre devolve `Dto`:**

```csharp
public class PedidoHandler : IHandler<GetPedidoByIdCommand, PedidoDto>
{
    public async Task<Result<PedidoDto>> Handle(GetPedidoByIdCommand command)
    {
        var pedido = await _pedidoRepository.ObterPorIdAsync(command.PedidoId);
        return Result<PedidoDto>.Success(PedidoDto.FromEntity(pedido));
    }
}
```

## 9. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Handler` retornando `Entity` em vez de `Dto` | Vaza modelo interno para fora do módulo |
| `Handler` com SQL embutido | SQL pertence exclusivamente ao `Repository` |
| `Handler` injetando `Repository`/`Service` de outro módulo | Viola isolamento — deve usar `I<NomeModulo>` (`Contracts`) |
| `Handler` conhecendo `HttpContext`/`IActionResult` | Acopla regra de aplicação ao transporte HTTP |
| `Handler` instanciando `IDbConnection`/`IDbTransaction` diretamente | Gerenciamento de conexão é responsabilidade de `IUnitOfWorkFactory` (`DATABASE/RULES.md` seção 5), não do Handler |
| Parâmetro booleano tipo `commit: bool` num método de `Repository` pra controlar se finaliza a transação | O controle de commit/rollback é sempre do `IUnitOfWork` (`.Commit()`/`Dispose()` automático), nunca de uma flag por chamada |
| Capturar exceção de infraestrutura só para virar `Result.Failure(Unexpected)` sem ação de recuperação | Duplica o que o middleware global já faz, esconde erro real do log de exceção |
| Um `Handler` implementando `IHandler<T,R>` para `Command`s de **substantivos diferentes** (ex: `GeneroHandler` também tratando `CreateAnimeCommand`) | Quebra a regra "banana vs. tomate" (seção 4) — cada Aggregate Root distinta tem seu próprio `Handler` |
| `switch`/`if` de tipo dentro de um único método `Handle` para tratar comandos diferentes | O ponto de sobrecarga é o próprio C# (múltiplos métodos `Handle`), nunca um dispatch manual dentro de um método genérico |
| `Handler` implementado corretamente, mas nenhum `Controller` o injeta — a rota HTTP chama um dispatcher genérico (`IMediator`/`ISender`/bus caseiro) em vez de `_handler.Handle(...)` | O `Handler` correto não compensa um `Controller` que reabriu a decisão "sem mediator" — ver `CONTROLLER/RULES.md` seções 3 e 8 para o anti-padrão e o teste de arquitetura que o pega |

## 10. Enforcement

- Code review confere que toda falha de negócio esperada retorna `Result.Failure` com o `ErrorType` correto, nunca exceção customizada para esse fim.
- Code review bloqueia qualquer `using` de namespace de outro módulo de negócio (`Entities`/`Repositories`/`Services` de outro módulo) dentro de um Handler — só `Shared.Contracts` é permitido, e fisicamente é o único que compila, já que não há `ProjectReference` para nenhum outro módulo (`CONTRACTS/RULES.md` seção 2).
- Code review confere que o construtor do `Handler` continua `public`, mesmo que suas dependências (`Repository` implementação, `Service`) sejam `internal` — só a interface injetada precisa ser visível (`ARCHITECTURE-RULES.md` seção 5.1).
- Code review aplica o teste "mesma Aggregate Root?" (seção 4) sempre que um `Handler` novo for proposto para um `Command` de um recurso que já tem `Handler` — decide se entra na classe existente ou vira uma nova.
