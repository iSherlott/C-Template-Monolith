# RULES — Modules / Repositories

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md`, `03-MODULES/RULES.md` e `02-INFRASTRUCTURE/DATABASE/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Repository` é a ponte entre a `Entity` (modelo de domínio puro) e o banco
(via Dapper, dentro do schema do próprio módulo). Ele não contém regra de
negócio — só sabe transformar uma `Entity` em linhas de SQL e linhas de
resultado de volta em `Entity`.

## 2. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Repositories/
    ├── Interface/
    │   └── IPedidoRepository.cs      # interface — public (ver visibilidade abaixo)
    └── Implementation/
        └── PedidoRepository.cs       # implementação concreta com Dapper — internal
```

- Mesma subpasta por tipo de arquivo (`Interface/`, `Implementation/`) já
  adotada em `Infrastructure` (`DATABASE/RULES.md` seção 3.1) — pelo mesmo
  motivo: legibilidade para quem nunca viu a pasta. Quem procura o
  contrato vai direto em `Interface/`; quem procura como o SQL é montado
  vai em `Implementation/`. Isso **não é** o mesmo que mover para
  `Contracts` (`CONTRACTS/RULES.md` seção 1.1) — `Interface/Implementation/`
  aqui é organização física de arquivo dentro do próprio `Repositories/`,
  nunca implica que `Repository` cruza a fronteira do módulo.
- Namespace continua `Modules.<NomeModulo>.Repositories` (ou
  `<NomeModulo>.Repositories`, seguindo a convenção sem prefixo composto)
  para os arquivos de ambas as subpastas — a divisão é só de arquivo, não
  de namespace, mesmo raciocínio de `DATABASE/RULES.md` seção 3.1.
- Interface e implementação **ambas privadas ao módulo** no sentido de que nenhum outro módulo as referencia — mas com visibilidades C# diferentes: a **interface é `public`** (aparece no construtor `public` do `Handler` — `ARCHITECTURE-RULES.md` seção 5.1) e a **implementação concreta é `internal`** (só é nomeada dentro do próprio `Install()`, nunca em assinatura pública). Isso não abre a fronteira — nenhum outro módulo tem `ProjectReference` para este módulo de qualquer forma (`CONTRACTS/RULES.md` seção 2).
- Um `Repository` por **Aggregate Root** (`ENTITIES/RULES.md` seção 4) — nunca um `Repository` para uma entidade filha (ex: não existe `IItemDoPedidoRepository`; itens são carregados/salvos como parte do `Pedido`).

## 3. Assinatura de método — `IUnitOfWork` explícito

Consistente com `DATABASE/RULES.md` seção 5: todo método de escrita recebe
`IUnitOfWork` explicitamente (nunca `IDbConnection`/`IDbTransaction` crus);
métodos de leitura pura podem abrir sua própria conexão via
`IDbConnectionFactory`.

```csharp
public interface IPedidoRepository
{
    Task InserirAsync(Pedido pedido, IUnitOfWork unitOfWork);
    Task AtualizarAsync(Pedido pedido, IUnitOfWork unitOfWork);
    Task<Pedido?> ObterPorIdAsync(Guid id);
    Task<bool> ExisteAsync(Guid id);
}

internal class PedidoRepository : IPedidoRepository
{
    private readonly IDbConnectionFactory _connectionFactory;

    public PedidoRepository(IDbConnectionFactory connectionFactory) =>
        _connectionFactory = connectionFactory;

    public Task InserirAsync(Pedido pedido, IUnitOfWork unitOfWork)
    {
        const string sql = """
            INSERT INTO vendas.pedidos (id, cliente_id, total, status)
            VALUES (@Id, @ClienteId, @Total, @Status)
            """;
        return unitOfWork.Connection.ExecuteAsync(sql, pedido, unitOfWork.Transaction);
    }

    public async Task<Pedido?> ObterPorIdAsync(Guid id)
    {
        using var connection = _connectionFactory.CreateConnection();
        const string sql = "SELECT id, cliente_id, total, status FROM vendas.pedidos WHERE id = @Id";
        return await connection.QueryFirstOrDefaultAsync<Pedido>(sql, new { Id = id });
    }
}
```

## 4. Reconstituindo uma `Entity` a partir do banco

`ENTITIES/RULES.md` define `Entity` com construtor privado e propriedades
`private set`. Isso **funciona com Dapper, mas não com o `CustomPropertyTypeMap`
padrão** — o mapper padrão do Dapper só busca construtor **público**. É por
isso que o type map global desta arquitetura é um `ReconstitutingTypeMap`
próprio, não o `CustomPropertyTypeMap` (`DATABASE/RULES.md` seção 7) — ele
busca o construtor também entre os não-públicos. Com esse type map registrado,
nenhum construtor ou setter público precisa ser exposto só para a persistência
funcionar; sem ele, toda `Query<Entity>` falha em runtime com
`"parameterless default constructor... is required"` no primeiro `GET` real.

Se uma `Entity` tiver uma coleção interna (ex: `Pedido.Itens`) que precisa
ser populada à parte (porque vem de uma segunda query, ex: itens do pedido),
o `Repository` monta isso explicitamente após a query principal — nunca
depende de mapeamento automático de coleção aninhada do Dapper.

## 5. Schema — sempre qualificado, sempre o do próprio módulo

Toda query referencia `<schema-do-modulo>.<tabela>` explicitamente
(`DATABASE/RULES.md` seção 6). Um `Repository` nunca faz `JOIN`/`SELECT`
contra schema de outro módulo — se precisar de dado de outro módulo, isso é
resolvido em camada de `Handler`/`Service` via `I<OutroModulo>`
(`ARCHITECTURE-RULES.md` seção 4), nunca dentro do `Repository`.

## 5.1 LINQ em vez de loop imperativo

`Repository` é a camada onde mais aparece transformação de coleção (linhas
de resultado de query → lista de `Entity`, montar itens de um agregado após
uma segunda query — seção 4 acima). Regra geral de estilo desta arquitetura
(`ARCHITECTURE-RULES.md` seção 7.1): usar LINQ (`Select`, `Where`, `ToList`,
...) em vez de `foreach` acumulando lista manualmente, sempre que não houver
efeito colateral por item (`await` sequencial, mutação externa) — nesse caso
`foreach` continua correto.

## 6. Sem regra de negócio aqui

`Repository` não valida nada além do estritamente técnico (ex: conexão
disponível). Toda validação de invariante já aconteceu dentro da `Entity`
antes dela chegar ao `Repository` para ser persistida — o `Repository` confia
que, se recebeu uma `Entity`, ela já está em estado válido.

## 7. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Repository` com regra de validação de negócio | Pertence à `Entity` (invariante) ou ao `Handler` (regra de aplicação) |
| `Repository` de entidade filha (`IItemDoPedidoRepository`) | Quebra o conceito de Aggregate Root |
| Método de escrita abrindo sua própria transação internamente | Contradiz o padrão de transação explícita controlada pelo `Handler` (`HANDLER/RULES.md` seção 3) |
| Query contra schema de outro módulo | Viola isolamento de módulo (`DATABASE/RULES.md` seção 6) |
| `SELECT *` ou SQL concatenado sem parâmetro | Ver `DATABASE/RULES.md` seção 8 |
| Uso de `Dapper.Contrib`/`Insert<T>` genérico | Perde controle explícito do SQL gerado — proibido nesta arquitetura |

## 8. Testando um `Repository` internal

`PedidoRepository` (implementação, `internal`) é instanciado diretamente nos
testes de Integration (`04-TEST/INTEGRATION/RULES.md`), que vivem num
projeto diferente (assembly diferente). Isso só compila porque o módulo
declara `[assembly: InternalsVisibleTo("<NomeModulo>.IntegrationTests")]`
(convenção: um arquivo `AssemblyInfo.cs` na raiz do módulo — ver
`04-TEST/INTEGRATION/RULES.md` seção sobre `internal`).

## 9. Enforcement

- Code review confere que todo método de escrita recebe `IUnitOfWork` explícito.
- Code review bloqueia qualquer query fora do schema do próprio módulo.
- Code review confere que a implementação concreta é `internal` e a interface é `public`.
