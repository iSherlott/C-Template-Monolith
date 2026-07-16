# RULES — Modules / Commands

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Commands` contém os objetos de entrada de um `Handler` — tanto comandos de
mutação quanto consultas (`Query`), conforme decidido em `03-MODULES/RULES.md`
seção 7. É o "pedido de trabalho" interno do módulo: dado puro, sem
comportamento, que descreve a intenção de quem está chamando o `Handler`.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Commands/
    ├── CriarPedidoCommand.cs
    ├── CancelarPedidoCommand.cs
    └── ObterPedidoPorIdQuery.cs
```

## 3. Forma — record imutável, sem comportamento

```csharp
public record CriarPedidoCommand(Guid ClienteId, IReadOnlyList<ItemPedidoInput> Itens);

public record ObterPedidoPorIdQuery(Guid PedidoId);
```

- Sempre `record` imutável — nunca classe com setters.
- Contém apenas dados primitivos ou tipos simples (`Guid`, `string`, `decimal`, coleções desses) necessários para o `Handler` processar. Nunca contém `Entity`.
- Sem métodos além do que o próprio `record` gera automaticamente (`Equals`, `ToString`). `Command`/`Query` não tem comportamento — é dado, não é a `Entity`.
- **Visibilidade sempre `public`**, mesmo não sendo consumido por outro módulo: é o tipo de parâmetro de ação do `Controller` (`public`, exigido pelo ASP.NET Core), então marcar `internal` quebraria a compilação por `CS0051` (`ARCHITECTURE-RULES.md` seção 5.1). "Privado ao módulo" (seção 5 abaixo) é isolamento arquitetural, não visibilidade C#.

## 4. Nomenclatura

| Tipo | Convenção | Exemplo |
|---|---|---|
| Command (mutação) | `<Verbo><Recurso>Command`, verbo no imperativo | `CriarPedidoCommand`, `CancelarPedidoCommand` |
| Query (consulta) | `<Verbo><Recurso>Query`, verbo de pergunta | `ObterPedidoPorIdQuery`, `ListarPedidosDoClienteQuery` |

## 5. Privado ao módulo — nunca é o mesmo tipo que atravessa a fronteira

`Command`/`Query` **não fica em `Contracts`** e nunca é o mesmo tipo usado
como `Request` de um `Controller` ou como `IntegrationEvent` recebido por um
`Consumer`. Quem cria o `Command` é sempre quem está do lado de dentro do
módulo, traduzindo uma entrada externa:

- `Controller` monta o `Command`/`Query` a partir do `Request` HTTP (`CONTROLLER/RULES.md` seção 3).
- `Consumer` monta o `Command` a partir do `IntegrationEvent` recebido de outro módulo (`CONSUMERS/RULES.md`).

Esse desacoplamento existe para que a forma externa (payload HTTP, payload de
evento) possa evoluir independentemente da forma que o `Handler` espera
internamente — mesmo que hoje as duas pareçam idênticas campo a campo.

## 6. Sem validação de forma aqui

Validação sintática (campo obrigatório, formato) já aconteceu na borda
(`Request` do `Controller`, `CONTROLLER/RULES.md` seção 6) antes do `Command`
ser criado. O `Command` em si não revalida isso — ele assume que, se existe,
foi construído a partir de uma entrada já validada em forma. Validação de
**regra de negócio** sobre o conteúdo do `Command` acontece dentro do
`Handler`, nunca dentro do próprio `Command`.

## 7. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Command`/`Query` mutável (setters públicos) | Deveria ser um valor imutável — é uma fotografia da intenção no momento da chamada |
| `Command` contendo `Entity` | Vaza modelo de domínio para dentro de um objeto de transporte interno |
| Método com lógica de negócio dentro do `Command` (ex: cálculo, validação de regra) | `Command` é dado; regra de domínio pertence à `Entity`, regra de aplicação ao `Handler` |
| `Command` reaproveitado como tipo de retorno do `Handler` | Retorno é sempre `Result<Dto>` (`HANDLER/RULES.md` seção 3), nunca o próprio `Command` |
| `Command` exposto em `Contracts` para outro módulo instanciar diretamente | Outro módulo só interage via `I<NomeModulo>` — nunca monta um `Command` interno de outro módulo |

## 8. Enforcement

- Code review bloqueia qualquer `Command`/`Query` com setter público ou método com lógica além dos gerados pelo `record`.
- Code review confere que nenhum `Command` é referenciado fora do próprio módulo.
