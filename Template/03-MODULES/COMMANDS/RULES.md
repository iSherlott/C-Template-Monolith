# RULES — Modules / Commands

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Commands` contém os objetos de entrada de um `Handler` — tanto mutação
quanto leitura, sempre sob o mesmo sufixo `Command` (seção 4), conforme
decidido em `03-MODULES/RULES.md` seção 7. É o "pedido de trabalho" interno
do módulo: dado puro, sem comportamento, que descreve a intenção de quem
está chamando o `Handler`.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Commands/
    ├── CreatePedidoCommand.cs
    ├── UpdatePedidoCommand.cs
    ├── CancelPedidoCommand.cs
    └── GetPedidoByIdCommand.cs
```

## 3. Forma — record imutável, sem comportamento

**❌ Errado:**

```csharp
public class CreatePedidoCommand // ❌ classe mutável, não record
{
    public Guid ClienteId { get; set; }             // ❌ setter público
    public Pedido? PedidoExistente { get; set; }     // ❌ Entity vazando pro Command
    public decimal CalcularTotal() => /* ... */;     // ❌ lógica de negócio dentro do Command
}
```

**✅ Correto:**

```csharp
public record CreatePedidoCommand(Guid ClienteId, IReadOnlyList<ItemPedidoInput> Itens);

public record GetPedidoByIdCommand(Guid PedidoId);
```

- Sempre `record` imutável — nunca classe com setters.
- Contém apenas dados primitivos ou tipos simples (`Guid`, `string`, `decimal`, coleções desses) necessários para o `Handler` processar. Nunca contém `Entity`.
- Sem métodos além do que o próprio `record` gera automaticamente (`Equals`, `ToString`). `Command` não tem comportamento — é dado, não é a `Entity`.
- **Visibilidade sempre `public`**, mesmo não sendo consumido por outro módulo: é o tipo de parâmetro de ação do `Controller` (`public`, exigido pelo ASP.NET Core), então marcar `internal` quebraria a compilação por `CS0051` (`ARCHITECTURE-RULES.md` seção 5.1). "Privado ao módulo" (seção 5 abaixo) é isolamento arquitetural, não visibilidade C#.

## 4. Nomenclatura — sufixo único `Command`, verbo em inglês identifica a ação

**Decisão:** não existe sufixo `Query` separado nesta arquitetura — toda
entrada de `Handler`, seja leitura ou escrita, termina em `Command`. O que
diferencia uma ação de outra é o **verbo em inglês** no início do nome, não
o sufixo. Motivo prático: um único `IHandler<TCommand, TResult>`
(`SHARED/RULES.md` seção 3) já cobre os dois casos sem distinção estrutural
— manter dois sufixos para a mesma forma de dado (record imutável, sem
comportamento) era diferença de nome sem diferença de comportamento.

| Ação | Verbo | Exemplo |
|---|---|---|
| Criar (mutação) | `Create` | `CreatePedidoCommand` |
| Atualizar (mutação) | `Update` | `UpdatePedidoCommand` |
| Consultar/ler (antigo "Query") | `Get` | `GetPedidoByIdCommand`, `GetPedidosByClienteCommand` |
| Remover (mutação) | `Delete` | `DeletePedidoCommand` |
| Ação de domínio que não é CRUD puro | Verbo de domínio em inglês | `CancelPedidoCommand`, `ConfirmPedidoCommand` |

- Os quatro verbos `Create`/`Update`/`Get`/`Delete` cobrem o caso comum (CRUD). Uma ação que não é CRUD puro (cancelar, confirmar, aprovar) usa o verbo de domínio em inglês que descreve a ação — sempre em inglês, nunca português, para manter consistência com os quatro verbos-base.
- `Get` cobre tanto "por Id" quanto listagens/filtros — o que muda é o restante do nome (`GetPedidoByIdCommand` vs. `GetPedidosByClienteCommand`), nunca o verbo.
- Nome sempre no imperativo/infinitivo em inglês + recurso + qualificador opcional (`ById`, `ByCliente`) + `Command`.

## 5. Privado ao módulo — nunca é o mesmo tipo que atravessa a fronteira

`Command` **não fica em `Contracts`** e nunca é o mesmo tipo usado como
`Request` de um `Controller` (`CONTRACTS/RULES.md` seção 1.2) ou como
`IntegrationEvent` recebido por um `Consumer`. Quem cria o `Command` é sempre
quem está do lado de dentro do módulo, traduzindo uma entrada externa:

- `Controller` monta o `Command` a partir do `Request` HTTP, ou direto de um parâmetro de rota/query string quando não há `Request` (`CONTROLLER/RULES.md` seção 3).
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
| `Command` mutável (setters públicos) | Deveria ser um valor imutável — é uma fotografia da intenção no momento da chamada |
| `Command` contendo `Entity` | Vaza modelo de domínio para dentro de um objeto de transporte interno |
| Método com lógica de negócio dentro do `Command` (ex: cálculo, validação de regra) | `Command` é dado; regra de domínio pertence à `Entity`, regra de aplicação ao `Handler` |
| `Command` reaproveitado como tipo de retorno do `Handler` | Retorno é sempre `Result<Dto>` (`HANDLER/RULES.md` seção 3), nunca o próprio `Command` |
| `Command` exposto em `Contracts` para outro módulo instanciar diretamente | Outro módulo só interage via `I<NomeModulo>` — nunca monta um `Command` interno de outro módulo |
| Sufixo `Query` separado, ou verbo em português (`Criar`/`Obter`/`Atualizar`/`Deletar`) | Reabre a decisão da seção 4 — sufixo é sempre `Command`, verbo é sempre em inglês |

## 8. Enforcement

- Code review bloqueia qualquer `Command` com setter público ou método com lógica além dos gerados pelo `record`.
- Code review confere que nenhum `Command` é referenciado fora do próprio módulo.
- Code review confere que todo `Command` novo segue o verbo em inglês da seção 4 (`Create`/`Update`/`Get`/`Delete` ou verbo de domínio em inglês), nunca um sufixo `Query` ou verbo em português.
