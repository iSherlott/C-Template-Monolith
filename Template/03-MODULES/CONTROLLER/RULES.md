# RULES — Modules / Controller

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Controller` é a camada de **tradução HTTP** — converte requisição em
`Command`/`Query`, chama o `Handler` correspondente, e converte o resultado de
volta em resposta HTTP. Não decide nada de negócio; só roteia e traduz.

Teste rápido: *"se eu trocasse o transporte de HTTP para gRPC amanhã, essa
linha de código sobreviveria?"* Se a resposta for não porque ela é regra de
negócio (não porque é sintaxe de protocolo), ela está no lugar errado.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Controller/
    └── <Recurso>Controller.cs       # ex: PedidosController.cs
```

Um `Controller` por recurso/agregado principal do módulo — não um único
`Controller` cobrindo múltiplos agregados sem relação direta.

## 3. Regra de composição — Handler injetado direto

Consistente com `ARCHITECTURE-RULES.md` seção 7 (sem mediator, resolução
manual via DI): o `Controller` injeta cada `Handler` que usa diretamente no
construtor, um por ação.

```csharp
[ApiController]
[Route("api/vendas/pedidos")]
public class PedidosController : ControllerBase
{
    private readonly CriarPedidoHandler _criarPedidoHandler;
    private readonly ObterPedidoPorIdHandler _obterPedidoHandler;

    public PedidosController(
        CriarPedidoHandler criarPedidoHandler,
        ObterPedidoPorIdHandler obterPedidoHandler)
    {
        _criarPedidoHandler = criarPedidoHandler;
        _obterPedidoHandler = obterPedidoHandler;
    }

    [HttpPost]
    public async Task<IActionResult> Post([FromBody] CriarPedidoRequest request)
    {
        var command = new CriarPedidoCommand(request.ClienteId, request.Itens);
        var result = await _criarPedidoHandler.Handle(command);
        return result.ToActionResult();
    }
}
```

- Proibido resolver `Handler` via `IServiceProvider.GetService(...)` (service locator) dentro do método de ação — a dependência é sempre explícita no construtor.
- Um método de ação chama **exatamente um** `Handler`. Se uma ação parece precisar orquestrar dois Handlers, a orquestração pertence a um `Handler`/`Service` novo, não ao Controller.

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
| Um `Controller` cobrindo mais de um agregado/recurso sem relação | Perde coesão, dificulta achar o dono de um endpoint |
| Mapeamento manual de `Result` para `IActionResult` reimplementado localmente | Diverge da tabela única da seção 5 |

## 8. Enforcement

- Code review bloqueia qualquer dependência de `Controller` que não seja um `Handler` do próprio módulo.
- Code review confere que toda ação usa `ToActionResult()` em vez de mapeamento manual.
