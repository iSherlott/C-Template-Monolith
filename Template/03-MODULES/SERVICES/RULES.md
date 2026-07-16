# RULES — Modules / Services

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Service` contém lógica de aplicação/domínio **reutilizável** dentro do
módulo — código chamado por mais de um `Handler`/`Consumer`, ou que
implementa a fachada pública `I<NomeModulo>` consumida por outros módulos.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Services/
    ├── <NomeModulo>ModuleFacade.cs    # implementação de I<NomeModulo> — ver CONTRACTS/RULES.md seção 3
    └── <Proposito>Service.cs          # ex: CalculadoraDeFreteService.cs
```

## 3. Quando algo vira `Service` (e quando não vira)

| Situação | Onde fica |
|---|---|
| Lógica usada por **um único** `Handler`/`Consumer` | Fica dentro do próprio `Handler`/`Consumer` — não extrai prematuramente |
| Lógica usada por **dois ou mais** `Handlers`/`Consumers` do mesmo módulo | Extrai para `Service` |
| Implementação da interface pública `I<NomeModulo>` | Sempre um `Service` (`<NomeModulo>ModuleFacade`), nunca implementada dentro de um `Handler` |
| Invariante de um único agregado (ex: "pedido não pode ter total negativo") | Fica na `Entity` (`ENTITIES/RULES.md`), nunca vira `Service` |

Regra prática: não criar `Service` "adiantado" prevendo reuso futuro — extrair
quando o segundo uso realmente aparecer. Duplicação de duas linhas por um
tempo é mais barata que uma abstração errada.

## 4. Diferença entre `Service` e `Handler`

- `Handler` é o ponto de entrada de **um** `Command`/`Query` específico — é chamado de fora do módulo (via `Controller`) ou traduzido de um evento (via `Consumer`).
- `Service` é reutilizável **dentro** do módulo — não tem um único "dono" de chamada; qualquer `Handler`/`Consumer` do mesmo módulo pode usá-lo.
- Um `Handler` pode chamar um `Service`; um `Service` nunca chama um `Handler` (evitaria ciclo de dependência e confusão sobre quem orquestra o quê).

## 5. `<NomeModulo>ModuleFacade` — a implementação de `I<NomeModulo>`

```csharp
internal class VendasModuleFacade : IVendasModule
{
    private readonly IPedidoRepository _pedidoRepository;

    public VendasModuleFacade(IPedidoRepository pedidoRepository) =>
        _pedidoRepository = pedidoRepository;

    public async Task<bool> ClienteTemPedidoAbertoAsync(Guid clienteId) =>
        await _pedidoRepository.ExisteAbertoParaClienteAsync(clienteId);
}
```

- Registrada no DI dentro do `Install()` de `<NomeModulo>ModuleInstaller` como a implementação de `I<NomeModulo>` (`CONTRACTS/RULES.md` seção 3).
- Só implementa os métodos que a interface pública declara — não é o lugar para lógica adicional que não faz parte do contrato exposto a outros módulos.
- **Visibilidade `internal`**: diferente de `IVendasModule` (que fica em `Shared/Contracts` e é `public`), a classe concreta nunca é nomeada fora do próprio `Install()` — nenhum consumidor a referencia diretamente, nem conseguiria (está em outro assembly). É um dos poucos pontos da arquitetura onde `internal` é aplicável de fato (`ARCHITECTURE-RULES.md` seção 5.1). `Service`s que não implementam `I<NomeModulo>` (ex: `CalculadoraDeFreteService`) seguem a mesma regra sempre que só são chamados de dentro do próprio módulo.

## 6. Transação — `Service` participa, não decide

Quando um `Service` é chamado a partir de um `Handler` que já está dentro de
um fluxo transacional (`HANDLER/RULES.md` seção 3), o `IUnitOfWork` é
passado para o `Service`, que repassa para o `Repository` — o `Service` nunca
abre sua própria transação por conta própria quando está participando de um
fluxo maior. Um `Service` chamado fora de um fluxo transacional (ex: pela
`ModuleFacade`, respondendo a outro módulo) pode operar sem transação quando
só faz leitura.

## 7. Lifetime e coesão

- Lifetime de registro: `Scoped` por padrão (`03-MODULES/RULES.md` seção 6). `Singleton` só quando o `Service` é comprovadamente sem estado mutável entre chamadas — decisão justificada em comentário no registro.
- Um `Service` tem **um propósito coeso**, nomeado pelo que faz (`CalculadoraDeFreteService`, `VendasModuleFacade`) — nunca um `PedidoService` genérico acumulando métodos não relacionados só porque "tem a ver com Pedido". Se um `Service` está crescendo em direções diferentes, é sinal de que deveria virar dois.

## 8. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Service` "genérico" acumulando métodos não relacionados (God Service) | Perde coesão, dificulta achar/testar lógica específica |
| `Service` chamando `Handler` | Inverte a direção de orquestração — confuso sobre quem manda em quem |
| `Service` acessando `Repository`/`Entity` de outro módulo | Viola isolamento — deve usar `I<OutroModulo>` |
| Lógica duplicada entre dois `Handlers` em vez de extraída para `Service` | Duplicação de regra de negócio, risco de divergência silenciosa |
| `Service` extraído antes de existir um segundo uso real | Abstração prematura sem necessidade comprovada |

## 9. Enforcement

- Code review pergunta, para todo `Service` novo: "isso é usado por mais de um lugar, ou é a `ModuleFacade`?" — se não, a lógica volta para dentro do `Handler`/`Consumer` que a usa.
