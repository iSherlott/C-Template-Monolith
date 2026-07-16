# RULES — Modules / Entities

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Entities` é o modelo de domínio do módulo — **totalmente privado**, nunca
atravessa a fronteira de `Contracts`. É aqui que a regra de negócio que
protege a consistência de um agregado vive, não no `Handler`.

Diferença de responsabilidade: `Handler` orquestra o fluxo de aplicação
(chamar Repository, publicar evento, decidir se o cliente tem permissão);
`Entity` protege seus próprios **invariantes** (um `Pedido` nunca pode ter
total negativo, um `Pedido` cancelado não pode receber novo item). Se uma
regra é sobre "o estado interno deste objeto pode ficar assim?", ela é da
`Entity`. Se é sobre "esse fluxo de aplicação pode prosseguir?", é do
`Handler`.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Entities/
    └── <NomeDaEntidade>.cs           # ex: Pedido.cs, ItemDoPedido.cs
```

## 3. Modelo rico, não anêmico

`Entity` nunca é um saco de propriedades com setters públicos. Ela expõe
comportamento nomeado com verbo de domínio; estado só muda através desses
métodos.

```csharp
public class Pedido : AggregateRoot // base definida em Shared/Kernel — SHARED/RULES.md
{
    public Guid ClienteId { get; private set; }
    public decimal Total { get; private set; }
    public StatusPedido Status { get; private set; }
    private readonly List<ItemDoPedido> _itens = new();
    public IReadOnlyCollection<ItemDoPedido> Itens => _itens.AsReadOnly();

    private Pedido() { } // uso exclusivo do Repository ao reconstituir a partir do banco

    public static Pedido Criar(Guid clienteId, IEnumerable<ItemDoPedido> itens)
    {
        if (!itens.Any())
            throw new DomainException("Pedido precisa ter ao menos um item.");

        var pedido = new Pedido { Id = Guid.NewGuid(), ClienteId = clienteId, Status = StatusPedido.Aberto };
        foreach (var item in itens)
            pedido.AdicionarItem(item);

        return pedido;
    }

    public void AdicionarItem(ItemDoPedido item)
    {
        if (Status != StatusPedido.Aberto)
            throw new DomainException("Não é possível adicionar item a um pedido que não está aberto.");

        _itens.Add(item);
        Total += item.Subtotal;
    }
}
```

- **Construtor privado + factory method estático** (`Criar`) — não existe `new Pedido()` público. Toda `Entity` só nasce em estado válido.
- Propriedades com `private set` (ou sem setter, só leitura). Mutação sempre por método nomeado com verbo de domínio (`AdicionarItem`, `Cancelar`, `Confirmar`) — nunca `pedido.Status = ...` de fora.
- Invariante de domínio violado lança `DomainException` (definida em `Shared/Kernel`) — isso é diferente de uma falha de negócio esperada tratada como `Result.Failure` no `Handler` (`HANDLER/RULES.md` seção 6): violação de invariante dentro da própria `Entity` é sinal de bug (alguém tentou colocar o objeto num estado impossível), não uma condição de negócio antecipada. O `Handler` decide se aquilo é esperado (e então nem chama o método) ou deixa a exceção propagar como erro real.

**Visibilidade:** apesar de "totalmente privada" ao módulo em termos de quem
a referencia (nenhum outro módulo importa uma `Entity`), a classe em si é
`public` em C#, não `internal`. Motivo: ela aparece na assinatura pública da
interface `Repository` (`REPOSITORIES/RULES.md` seção 3), que por sua vez
aparece no construtor público do `Handler` — marcar `Entity` como `internal`
quebraria a compilação por `CS0051` (`ARCHITECTURE-RULES.md` seção 5.1).
"Privada ao módulo" aqui é uma garantia de isolamento arquitetural (imposta
por ninguém ter `ProjectReference` para este módulo), não de visibilidade
C#.

## 4. Identidade e Aggregate Root

- `Id` é sempre `Guid`, gerado no cliente (na `Entity`, dentro do factory method), nunca gerado pelo banco (`IDENTITY`/`SEQUENCE`). Mantém consistência entre o momento em que o objeto existe em memória e quando é persistido, e facilita geração de `Id` antes mesmo do primeiro `INSERT` (necessário, por exemplo, para incluir no payload de um `IntegrationEvent` publicado na mesma transação).
- Um módulo pode ter várias `Entities`, mas só a **Aggregate Root** (ex: `Pedido`) tem `Repository` próprio. Entidades filhas (ex: `ItemDoPedido`) só são acessadas através da raiz (`pedido.Itens`) — nunca têm `Repository` nem são carregadas/salvas independentemente.
- `AggregateRoot`/`Entity` (classes base) vêm de `Shared/Kernel` — todo agregado do sistema herda da mesma base, garantindo campos comuns (`Id`, talvez `CreatedAt`) e mecanismo comum de igualdade por identidade.

## 5. Persistence ignorance

`Entity` não sabe que está sendo persistida por Dapper. Nunca contém:

- Atributos de mapeamento (`[Table]`, `[Column]`, `[Key]`) — mapeamento é responsabilidade do `Repository` (`REPOSITORIES/RULES.md`), que monta a `Entity` manualmente a partir do resultado da query ou usa reflection controlada apenas para reconstituição, nunca a `Entity` decorada para isso.
- Qualquer `using` de `System.Data`, Dapper, ou qualquer tipo de `Infrastructure`.
- Qualquer referência a `Dto` — `Entity` não sabe que `Contracts` existe; é o `Handler` que faz essa ponte (`HANDLER/RULES.md` seção 7).

## 6. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Propriedade com `set` público | Permite estado inválido ser atribuído de fora, sem passar pela regra de domínio |
| `Entity` anêmica (só propriedades, toda regra no `Handler`) | Perde a proteção de invariante no lugar certo — regra de domínio espalhada e duplicável |
| Atributo de Dapper/EF na `Entity` | Acopla modelo de domínio à tecnologia de persistência |
| `Entity` filha com `Repository` próprio | Quebra o conceito de Aggregate Root — consistência do agregado inteiro fica sem dono |
| `Entity` referenciando `Dto` ou tipo de `Contracts` | Inverte a dependência — quem conhece quem é sempre `Handler`/`Dto` conhecendo `Entity`, nunca o contrário |
| `Id` gerado pelo banco (`IDENTITY`) | Impede ter o `Id` disponível antes do `INSERT`, necessário para eventos publicados na mesma transação |

## 7. Enforcement

- Code review bloqueia qualquer setter público em propriedade de `Entity` e qualquer `using` de namespace de `Infrastructure`/Dapper dentro de `Entities/`.
- Code review confere que toda criação de `Entity` passa por um factory method, nunca por construtor público.
