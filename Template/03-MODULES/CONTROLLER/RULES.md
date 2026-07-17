# RULES — Modules / Controller

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Controller` é a camada de **tradução HTTP** — converte requisição em
`Command` (mutação ou leitura — `COMMANDS/RULES.md` seção 4), chama o
`Handler` correspondente, e converte o resultado de volta em resposta HTTP.
Não decide nada de negócio; só roteia e traduz.

Teste rápido: *"se eu trocasse o transporte de HTTP para gRPC amanhã, essa
linha de código sobreviveria?"* Se a resposta for não porque ela é regra de
negócio (não porque é sintaxe de protocolo), ela está no lugar errado.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Controller/
    └── <Recurso>Controller.cs       # ex: PedidosController.cs
```

Um `Controller` por área de recurso do módulo — pode cobrir mais de um
Aggregate Root quando eles compartilham o mesmo prefixo de rota lógico (ex:
`CatalogoController` responde por `api/catalogo/generos` e
`api/catalogo/animes`, dois Aggregate Roots diferentes sob o mesmo módulo).
O que o `Controller` nunca faz é misturar recursos **sem relação** só por
conveniência — `HANDLER/RULES.md` seção 4 tem o critério de quando isso
deveria ser dois `Controller`s (e dois `Handler`s) em vez de um.

## 3. Regra de composição — Handler injetado direto, um por Aggregate Root

Consistente com `ARCHITECTURE-RULES.md` seção 7 (sem mediator, resolução
manual via DI): o `Controller` injeta o `Handler` de cada Aggregate Root que
expõe — **um único `Handler` cobre todas as ações daquele recurso**
(`HANDLER/RULES.md` seção 3), não um `Handler` por ação.

**❌ Errado — dispatcher genérico (MediatR ou qualquer bus/dispatcher caseiro
com método `Send`/`SendAsync`/`Dispatch`) resolvendo o handler por reflection
em runtime:**

```csharp
[ApiController]
[Route("api/vendas/pedidos")]
public class PedidosController : ControllerBase
{
    private readonly IMediator _mediator; // ou ISender, IDispatcher, IRequestBus — mesmo problema

    public PedidosController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] CreatePedidoRequest request)
    {
        var command = new CreatePedidoCommand(request.ClienteId, request.Itens);
        var result = await _mediator.SendAsync(command); // ❌ proibido — ver ARCHITECTURE-RULES.md §7
        return Ok(result);
    }
}
```

Este padrão está banido **mesmo que o `Handler` correspondente exista e
esteja corretamente implementado** — o problema não é o `Handler`, é o
`Controller` nunca chamar `Handle()` nele diretamente. Se existe uma classe
`PedidoHandler` no módulo mas o `Controller` não a injeta, o `Handler` é
código morto do ponto de vista de roteamento real, e a garantia de "ir para
definição" (compilador resolve o overload, não reflection) deixa de existir.

**✅ Correto — dependência explícita no construtor, chamada direta a `Handle()`:**

```csharp
[ApiController]
[Route("api/vendas/pedidos")]
public class PedidosController : ControllerBase
{
    private readonly PedidoHandler _pedidoHandler;

    public PedidosController(PedidoHandler pedidoHandler) => _pedidoHandler = pedidoHandler;

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] CreatePedidoRequest request)
    {
        var command = new CreatePedidoCommand(request.ClienteId, request.Itens);
        var result = await _pedidoHandler.Handle(command);
        return result.ToActionResult(StatusCodes.Status201Created);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> Get(Guid id)
    {
        var result = await _pedidoHandler.Handle(new GetPedidoByIdCommand(id));
        return result.ToActionResult();
    }
}
```

Se o `Controller` cobre mais de um Aggregate Root (seção 2), injeta um
`Handler` por Aggregate Root — nunca um único `Handler` genérico tratando
substantivos diferentes (`HANDLER/RULES.md` seção 4, critério "banana vs.
tomate").

- Proibido resolver `Handler` via `IServiceProvider.GetService(...)` (service locator) dentro do método de ação — a dependência é sempre explícita no construtor.
- Proibido qualquer tipo injetado cujo único papel seja rotear uma mensagem para o handler certo em runtime (`IMediator`, `ISender`, ou um nome caseiro como `IRequestDispatcher`/`ICommandBus`) — o roteamento é sempre resolvido em tempo de compilação pelo overload de `Handle()`, nunca em tempo de execução por reflection/nome de tipo.
- Um método de ação chama **exatamente um** `Handle` de **um** `Handler`. Se uma ação parece precisar orquestrar dois Handlers de Aggregate Roots diferentes, a orquestração pertence a um `Service`/evento assíncrono, não ao Controller.

## 4. Rota e nomenclatura

- Prefixo de rota sempre `api/<schema-do-modulo>/<recurso>` — usa o mesmo nome de schema definido em `03-MODULES/RULES.md` seção 3. Ex: módulo `Vendas`, schema `vendas` → `api/vendas/pedidos`.
- Sem versionamento de URL por padrão (`api/v1/...`) nesta fase — se/quando houver necessidade real de versionar um endpoint, essa decisão é adicionada aqui explicitamente, não antecipada sem caso concreto.
- Nome da classe: `<Recurso>Controller`, plural do recurso (ex: `PedidosController`, não `PedidoController`).

## 5. Tradução de `Result<T>` para HTTP

Todo `Handler` retorna um `Result<TDto>` (primitivo definido em
`SHARED/RULES.md` — Kernel). O `Controller` nunca inspeciona esse resultado
manualmente com `if` — usa a extensão `ToActionResult()` (vive em
`Modules/Shared/Web`) que padroniza o mapeamento para toda a aplicação:

| Estado do `Result` | Status HTTP |
|---|---|
| Sucesso, com valor, ação de criação (`POST`) | `201 Created` |
| Sucesso, com valor, ação de leitura (`GET`) | `200 OK` |
| Sucesso, sem valor (`PUT`/`DELETE` que não retornam corpo) | `204 No Content` |
| Falha — `Validation` | `400 Bad Request` |
| Falha — `NotFound` | `404 Not Found` |
| Falha — `Conflict` | `409 Conflict` |
| Falha — `Unauthorized`/`Forbidden` | `401`/`403` |
| Falha — `Unexpected` | `500 Internal Server Error` |

Essa tabela é a única fonte de verdade do mapeamento — nenhum Controller
implementa esse `switch` localmente.

## 6. Validação — o que cabe aqui vs. o que cabe no Handler

- **Validação sintática/de forma** (campo obrigatório, tipo, tamanho máximo) acontece na borda: atributos de data annotation no `Request` (`[Required]`, `[MaxLength]`) combinados com `[ApiController]`, que já retorna `400` automaticamente em `ModelState` inválido. Isso é aceitável no Controller porque é sobre o **formato** da requisição, não sobre uma regra do domínio.
- **Validação de regra de negócio** (ex: "cliente precisa ter limite de crédito disponível") nunca acontece no Controller — é responsabilidade do `Handler`/`Service`, expressa como uma falha `Validation` ou `Conflict` no `Result` retornado.

## 7. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Controller injetando/chamando `Repository` ou `Service` diretamente | Pula o ponto único de orquestração (`Handler`) |
| `if`/`switch` de regra de negócio dentro de uma ação | Regra de domínio pertence ao `Handler` |
| Controller retornando `Entity` serializada diretamente | Vaza modelo interno — deve sempre retornar `Dto` de `Contracts` |
| `Handler` resolvido via `IServiceProvider.GetService` dentro do método | Esconde a dependência real, quebra o padrão de DI explícita |
| Controller injetando `IMediator`/`ISender` (MediatR) ou qualquer dispatcher caseiro (`IRequestDispatcher`, `ICommandBus`, método `Send`/`SendAsync`/`Dispatch` genérico) em vez do `Handler` concreto | Reabre a decisão já fechada em `ARCHITECTURE-RULES.md` §7 (sem mediator) — troca resolução em tempo de compilação por reflection em runtime, mesmo que o `Handler` correto exista no módulo (seção 3 acima) |
| Um `Controller` cobrindo mais de um agregado/recurso sem relação | Perde coesão, dificulta achar o dono de um endpoint |
| Mapeamento manual de `Result` para `IActionResult` reimplementado localmente | Diverge da tabela única da seção 5 |

## 8. Enforcement

- Code review bloqueia qualquer dependência de `Controller` que não seja um `Handler` do próprio módulo.
- Code review confere que toda ação usa `ToActionResult()` em vez de mapeamento manual.
- **Automatizado, `Test/Architecture/`:** teste de arquitetura (mesmo mecanismo de `04-TEST/CONTRACT/RULES.md` seção 3) valida que o namespace `Controller` de nenhum módulo depende de `MediatR`/`IMediator`/`ISender`, nem de qualquer assembly de terceiro cujo papel seja dispatch genérico:

  ```csharp
  [Theory]
  [MemberData(nameof(TodosOsModulos))]
  public void Controllers_NaoDevemDepender_DeMediatorOuDispatcherGenerico(string moduloNome)
  {
      var resultado = Types.InAssembly(Assembly.Load(moduloNome))
          .That().ResideInNamespace($"{moduloNome}.Controller")
          .Should()
          .NotHaveDependencyOnAny("MediatR", "IMediator", "ISender")
          .GetResult();

      Assert.True(resultado.IsSuccessful, resultado.FailingTypes is null
          ? "OK"
          : string.Join(", ", resultado.FailingTypeNames));
  }
  ```

  Se o projeto tem um dispatcher caseiro (nome diferente de `MediatR`), adiciona o nome do assembly/namespace dele na lista de `NotHaveDependencyOnAny` — o teste existe para qualquer variante do padrão, não só a biblioteca MediatR especificamente. Este teste entra na mesma suíte `Test/Architecture` descrita em `04-TEST/CONTRACT/RULES.md` seção 3.
