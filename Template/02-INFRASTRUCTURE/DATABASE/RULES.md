# RULES — Infrastructure / Database

> Herda todas as regras de [`00-PRINCIPLES/ARCHITECTURE-RULES.md`](../../00-PRINCIPLES/ARCHITECTURE-RULES.md). Este documento especializa, nunca substitui.

## 1. Missão

Esta camada fornece o **acesso técnico** ao banco de dados via Dapper — nada
de regra de domínio, nada de `Entity` de módulo algum. `Infrastructure/Database`
não sabe o que é um "Pedido" ou um "Produto"; ela só sabe abrir conexão,
executar SQL parametrizado e devolver dados.

`Repository` é código de módulo (`Modules/<Nome>/Repositories/`) — ele
**consome** o que esta camada expõe. Esta camada nunca implementa um
Repository de negócio, só o que é genérico e reutilizável por qualquer módulo.

## 2. Decisões técnicas desta camada

| Decisão | Escolha |
|---|---|
| Banco de dados | **SQL Server** |
| Ferramenta de migration | **DbUp** (scripts `.sql` versionados, sem modelo C# de schema) |
| Convenção de nomenclatura no banco | **snake_case** (`pedido_id`, `criado_em`) — propriedades C# continuam PascalCase |
| Transação / Unit of Work | **`IUnitOfWork`**, criado por `IUnitOfWorkFactory` e repassado a cada Repository chamado no mesmo fluxo |
| Isolamento entre módulos | **Schema por módulo** (ver `ARCHITECTURE-RULES.md` seção 6) |

## 3. Estrutura de pastas

`Database`, assim como `Messaging` e `Cache`, é uma **subpasta dentro do
projeto único `Infrastructure`** (`Infrastructure/Infrastructure.csproj`) —
não um `.csproj` próprio (`ARCHITECTURE-RULES.md` seção 5, "Granularidade de
assembly por módulo"; a mesma decisão vale para `Infrastructure`). O
namespace correspondente é `Infrastructure.Database`.

```
Infrastructure/
└── Database/
    ├── Interface/
    │   ├── IDbConnectionFactory.cs
    │   ├── IUnitOfWork.cs
    │   └── IUnitOfWorkFactory.cs
    ├── Factory/
    │   ├── SqlServerConnectionFactory.cs
    │   ├── UnitOfWork.cs            # implementação — só é instanciada de dentro de UnitOfWorkFactory
    │   └── UnitOfWorkFactory.cs
    ├── Map/
    │   ├── SnakeCaseTypeMap.cs      # configuração do mapeamento Dapper (seção 7)
    │   └── ReconstitutingTypeMap.cs
    ├── Extensions/
    │   └── DatabaseExtensions.cs    # AddDatabase(IServiceCollection, IConfiguration)
    └── Migrations/
        ├── Runner/                  # bootstrap do DbUp (aplicação dos scripts)
        └── Scripts/
            ├── vendas/               # scripts .sql do schema do módulo Vendas
            │   ├── 0001_create_schema_vendas.sql
            │   ├── 0002_create_table_pedidos.sql
            │   └── ...
            ├── estoque/
            │   └── ...
            └── <schema-do-modulo>/
```

### 3.1 Por que subpastas por tipo (`Interface/`, `Factory/`, `Map/`, `Extensions/`)

Igual à decisão de estilo de código (`ARCHITECTURE-RULES.md` seção 7.1), a
prioridade é ser **humanamente intuitivo** para quem nunca viu esta pasta:
quem procura "onde está o contrato de X" vai direto em `Interface/`; quem
procura "onde X é de fato criado/aberto" vai em `Factory/`. Sem essa divisão,
uma pasta com 8+ arquivos soltos (interface, implementação, factory,
extensão de DI, type map, tudo junto) exige ler o nome de cada arquivo um
por um para entender o que é o quê.

- `Interface/` — só contratos (`I<Algo>.cs`), nunca uma implementação.
- `Factory/` — quem cria/abre o recurso técnico (`SqlServerConnectionFactory`,
  `UnitOfWorkFactory`) e, junto dela, a implementação que só existe como
  produto dessa fábrica (`UnitOfWork` só é instanciado dentro de
  `UnitOfWorkFactory.Create()` — `DATABASE/RULES.md` seção 5 — não faz
  sentido físico separá-los em pastas diferentes).
- `Map/` — configuração de mapeamento objeto-relacional (type maps do Dapper).
- `Extensions/` — método de extensão `Add<Algo>(IServiceCollection, IConfiguration)`
  chamado pelo Host/outro `Install()`.
- `Migrations/` continua **fora** dessa divisão por tipo — é uma área de
  funcionalidade coesa (scripts + runner), não um tipo de arquivo técnico;
  fragmentá-la em `Interface/Factory/...` só pra seguir a convenção à risca
  destruiria a organização que já faz sentido nela. A mesma lógica se aplica
  a `RabbitMq/` e `Outbox/` dentro de `Messaging` (seção 3 de
  `MESSAGING/RULES.md`) — pastas de funcionalidade coesa e pequena mantêm sua
  própria organização interna, sem subdividir ainda mais por tipo de arquivo.
- **Namespace continua `Infrastructure.Database` para todo arquivo dentro de
  `Interface/`/`Factory/`/`Map/`/`Extensions/`, independente da subpasta**
  — em C#, namespace não é obrigado a seguir a pasta física. Essas quatro
  subpastas são só organização de arquivo (uma "gaveta" no explorador),
  não uma fronteira de contrato — mudar o namespace por subpasta obrigaria
  todo `GlobalUsings.cs` de módulo (`03-MODULES/RULES.md` seção 2) a listar
  `Infrastructure.Database.Interface`, `.Factory`, `.Map` separadamente sem
  ganho real. `Migrations/Runner/` é a exceção deliberada — tem namespace
  próprio (`Infrastructure.Database.Migrations.Runner`) porque é uma área
  de funcionalidade com identidade própria, não um tipo de arquivo.
- Se um arquivo misturar interface e implementação no mesmo `.cs` (ex:
  `IUnitOfWorkFactory.cs` continha tanto a interface quanto a classe
  `UnitOfWorkFactory` antes desta reorganização), ele é **separado em dois
  arquivos** na hora de mover — um por subpasta, nunca um arquivo só
  fisicamente duplicado ou um `.cs` pertencendo a duas pastas ao mesmo tempo.
- Regra prática para decidir se uma subpasta nova (`Config/`, por exemplo)
  se justifica: só criar quando houver **dois ou mais arquivos** daquele
  tipo — uma pasta com um único arquivo dentro não ganha nada em
  legibilidade, só adiciona um nível de navegação a mais.

## 4. `IDbConnectionFactory` — contrato

```csharp
public interface IDbConnectionFactory
{
    IDbConnection CreateConnection();
}
```

- Cada chamada a `CreateConnection()` retorna uma conexão **nova e fechada**; quem chama é responsável por `Open()`/`using`/dispose. Nunca uma conexão singleton de longa duração.
- Só existe uma implementação concreta (`SqlServerConnectionFactory`), registrada em `AddDatabase()`. Módulos nunca instanciam `SqlConnection` diretamente — sempre via `IDbConnectionFactory` injetado.
- A connection string vem de `Infrastructure:Database:ConnectionString` (lida dentro de `AddDatabase()`, nunca repassada manualmente pelo Host — ver `01-HOST/RULES.md` seção 4).

## 5. Transação — `IUnitOfWork`

**O `Handler` nunca toca `IDbConnection`/`IDbTransaction` diretamente.** Abrir
e fechar a conexão é responsabilidade da camada de persistência
(`Infrastructure/Database`), não do Handler — o Handler só pede um
`IUnitOfWork` pronto e decide quando confirmar (`Commit()`) ou deixar
reverter.

```csharp
public interface IUnitOfWork : IDisposable
{
    IDbConnection Connection { get; }
    IDbTransaction Transaction { get; }
    void Commit();
    void Rollback();
}

public interface IUnitOfWorkFactory
{
    IUnitOfWork Create();
}

public class UnitOfWork : IUnitOfWork
{
    private bool _finalizado;

    internal UnitOfWork(IDbConnection connection, IDbTransaction transaction)
    {
        Connection = connection;
        Transaction = transaction;
    }

    public IDbConnection Connection { get; }
    public IDbTransaction Transaction { get; }

    public void Commit()
    {
        Transaction.Commit();
        _finalizado = true;
    }

    public void Rollback()
    {
        Transaction.Rollback();
        _finalizado = true;
    }

    public void Dispose()
    {
        if (!_finalizado)
            Transaction.Rollback(); // rollback automático se ninguém commitou — cobre o caso de exceção no meio do processo

        Transaction.Dispose();
        Connection.Dispose();
    }
}

public class UnitOfWorkFactory : IUnitOfWorkFactory
{
    private readonly IDbConnectionFactory _connectionFactory;

    public UnitOfWorkFactory(IDbConnectionFactory connectionFactory) => _connectionFactory = connectionFactory;

    public IUnitOfWork Create()
    {
        var connection = _connectionFactory.CreateConnection();
        connection.Open();
        var transaction = connection.BeginTransaction();
        return new UnitOfWork(connection, transaction);
    }
}
```

Uso no `Handler` (ver `HANDLER/RULES.md` seção 5 para o padrão completo):

```csharp
using var unitOfWork = _unitOfWorkFactory.Create();

await _pedidoRepository.InserirAsync(pedido, unitOfWork);
await _estoqueLocalRepository.DebitarAsync(item, unitOfWork); // mesmo unitOfWork — mesma transação

unitOfWork.Commit();
```

- **Sem parâmetro booleano por chamada** decidindo "commita agora ou não" — o mesmo `IUnitOfWork` é reaproveitado em quantas chamadas de `Repository` o `Handler` precisar (inclusive entre Repositories diferentes, como no exemplo acima), e só um `Commit()` explícito, no final, decide o destino de todas juntas.
- **Rollback é automático**: se uma exceção interrompe o `Handler` antes do `Commit()`, o `using` chama `Dispose()`, que reverte a transação sozinho — não é necessário `try/catch` manual para isso.
- `IUnitOfWorkFactory` é `Singleton` (não tem estado próprio, só fabrica objetos) — pode ser injetado tanto em `Handler` (`Scoped`) quanto em `Consumer` (`Singleton`, ver `CONSUMERS/RULES.md` seção 3.1) sem o problema de "Singleton consumindo Scoped".
- Todo método de `Repository` que grava dado recebe `IUnitOfWork unitOfWork` como parâmetro explícito (usa `unitOfWork.Connection`/`unitOfWork.Transaction` internamente) — nunca abre sua própria conexão internamente para uma operação de escrita.
- Métodos de leitura pura (`Query`, `QueryFirstOrDefaultAsync`) podem abrir e fechar sua própria conexão via `IDbConnectionFactory` quando não fazem parte de um fluxo transacional maior.
- Um `Handler` nunca cria `IUnitOfWork` "por garantia" quando só grava em um único Repository — o custo de criar é baixo, mas simplicidade continua sendo o padrão: se não há necessidade real de atomicidade entre múltiplas escritas, ainda assim usa-se `IUnitOfWork` (é o único caminho que `Repository` aceita), só não se multiplica chamadas desnecessárias.

## 6. Schema por módulo — aplicação prática

- Toda query dentro de um `Repository` referencia o schema do próprio módulo, qualificado explicitamente no SQL: `SELECT * FROM vendas.pedidos WHERE ...`. Nunca depende do `default schema` da conexão.
- Nenhuma query dentro de `Modules/<Nome>/Repositories/` pode referenciar `FROM <outro-schema>.<tabela>`. Isso é validado no code review e é candidato a regra automatizada (grep/lint de padrão `FROM \w+\.` comparando contra o schema esperado do módulo) — a formalizar quando o CI for definido.
- A criação do schema (`CREATE SCHEMA vendas`) é o primeiro script de migration do módulo (`0001_create_schema_<modulo>.sql`).

## 7. Convenção de nomenclatura — snake_case no banco, PascalCase no C#

Dapper não converte `snake_case` ↔ `PascalCase` automaticamente. Esta
arquitetura resolve isso com **um type map customizado, registrado uma única
vez, globalmente, dentro de `AddDatabase()`**.

**Não usar `Dapper.CustomPropertyTypeMap`** para isso — seu `FindConstructor`
interno só enxerga construtores **públicos**. Como toda `Entity` desta
arquitetura tem construtor parameterless **privado** (`ENTITIES/RULES.md`
seção 3, usado exclusivamente para reconstituição pelo `Repository`),
`CustomPropertyTypeMap` falha em tempo de execução com
`"A parameterless default constructor... is required"` assim que uma query
tenta materializar uma `Entity` — descoberto rodando o primeiro `GET` real
contra o banco, não em tempo de compilação. Em vez disso, implementa-se um
`SqlMapper.ITypeMap` próprio que busca o construtor também entre os
não-públicos:

Uma segunda armadilha: `Repository` também usa `Query<T>` para **records de
projeção de leitura** (ex: um resultado de agregação `record
PessoaComAnimesEmComum(Guid PessoaId, int AnimesEmComum)`, sem construtor
parameterless — só o construtor posicional). Se `FindConstructor` sempre
procurar *apenas* o parameterless (mesmo incluindo os privados), esses
records de projeção quebram com o mesmo erro. A implementação correta tenta
**primeiro** um construtor posicional casando por parâmetro (cobre records de
projeção) e só então cai para o parameterless — inclusive privado (cobre
`Entity`):

```csharp
public class ReconstitutingTypeMap : SqlMapper.ITypeMap
{
    private readonly Type _type;
    private readonly Func<Type, string, PropertyInfo?> _propertySelector;

    public ReconstitutingTypeMap(Type type, Func<Type, string, PropertyInfo?> propertySelector)
    {
        _type = type;
        _propertySelector = propertySelector;
    }

    public ConstructorInfo? FindConstructor(string[] names, Type[] types)
    {
        var posicional = _type
            .GetConstructors(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic)
            .FirstOrDefault(ctor =>
            {
                var parametros = ctor.GetParameters();
                return parametros.Length == names.Length
                    && parametros.All(p => names.Any(n => ColumnMatches(n, p.Name!)));
            });

        return posicional ?? _type.GetConstructor(
            BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic, null, Type.EmptyTypes, null);
    }

    public ConstructorInfo? FindExplicitConstructor() =>
        _type.GetConstructor(BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic, null, Type.EmptyTypes, null);

    public SqlMapper.IMemberMap? GetConstructorParameter(ConstructorInfo constructor, string columnName)
    {
        var parametro = constructor.GetParameters().FirstOrDefault(p => ColumnMatches(columnName, p.Name!));
        return parametro is null ? null : new ParameterMemberMap(columnName, parametro);
    }

    public SqlMapper.IMemberMap? GetMember(string columnName)
    {
        var property = _propertySelector(_type, columnName);
        return property is null ? null : new PropertyMemberMap(columnName, property);
    }

    private static bool ColumnMatches(string columnName, string memberName) =>
        string.Equals(columnName.Replace("_", ""), memberName, StringComparison.OrdinalIgnoreCase);

    // PropertyMemberMap e ParameterMemberMap: implementações mínimas de SqlMapper.IMemberMap
    // envolvendo um PropertyInfo/ParameterInfo (Dapper não expõe SimpleMemberMap publicamente).
}
```

```csharp
public static class SnakeCaseTypeMap
{
    public static void Register()
    {
        SqlMapper.TypeMapProvider = type => new ReconstitutingTypeMap(type, (t, columnName) =>
            t.GetProperties().FirstOrDefault(p =>
                string.Equals(p.Name, ToPascalCase(columnName), StringComparison.OrdinalIgnoreCase)));
    }

    private static string ToPascalCase(string snakeCase) =>
        string.Concat(snakeCase.Split('_').Select(s => char.ToUpperInvariant(s[0]) + s[1..]));
}
```

- A propriedade em si (`Id`, `Nome`, ...) continua **pública** — só o `set` é privado (`ENTITIES/RULES.md` seção 3) e só o construtor é privado. Reflection via `PropertyInfo.SetValue` ignora a visibilidade do `set` normalmente, então esse lado sempre funcionou; o problema real e não-óbvio estava só na escolha do construtor.
- Nenhum módulo escreve seu próprio mapeamento de coluna — o type map global cobre todos. Se uma classe precisar de um mapeamento especial (ex: coluna com nome muito diferente da propriedade), o ajuste é feito com `[Column("nome_da_coluna")]`/alias no `SELECT ... AS`, nunca criando um segundo type map paralelo.
- Colunas em queries são sempre escritas em `snake_case`; propriedades de DTO/Entity em código são sempre `PascalCase`. Nunca misturar as duas convenções dentro do mesmo arquivo `.sql` ou classe.

## 8. Regras de escrita de query

- SQL sempre **parametrizado** (`@Parametro`) — nunca concatenação de string com valor vindo de fora. Isso é regra de segurança (SQL Injection), não estilo.
- Nunca `SELECT *`. Sempre listar as colunas explicitamente — protege contra breaking change silencioso quando alguém adiciona coluna nova na tabela.
- Query como string inline no método do Repository é aceitável para queries curtas; queries longas/complexas podem virar constante `private const string` no topo da classe do Repository, mas continuam dentro do módulo — nunca em arquivo `.sql` solto fora de `Migrations/`.
- Sem ORM de mapeamento automático de escrita (nada de `Dapper.Contrib`, `Insert<T>`/`Update<T>` genérico) — todo `INSERT`/`UPDATE`/`DELETE` é SQL explícito escrito à mão no Repository. Mantém o controle total sobre o SQL gerado, coerente com a escolha de Dapper puro sobre um ORM completo.

## 9. Migrations com DbUp

- Scripts `.sql` numerados sequencialmente por módulo (`0001_`, `0002_`, ...), nunca reordenados ou editados após terem sido aplicados em qualquer ambiente compartilhado — correção de erro em script já aplicado é sempre um script novo, nunca uma edição retroativa.
- O `Runner` (dentro de `Migrations/Runner/`) aplica os scripts de todos os módulos numa única execução do DbUp, na ordem: schema-first (scripts `0001_create_schema_*`) antes de qualquer tabela.
- Aplicação de migration é um **passo de deploy separado**, não parte do `Program.cs` do Host em produção — evita múltiplas instâncias do serviço tentando migrar o schema simultaneamente na subida. Em ambiente de desenvolvimento local, é aceitável um atalho que roda o `Runner` automaticamente no startup, guardado por checagem explícita de ambiente (`if (env.IsDevelopment())`).

## 10. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Repository` de um módulo fazendo `SELECT` em schema de outro módulo | Viola isolamento de módulo tanto quanto acessar `Entity` de outro módulo em código |
| Conexão `SqlConnection` de longa duração guardada em campo estático/singleton | Esgota pool de conexão, mascara vazamento de recursos |
| Concatenação de valor de usuário direto na string SQL | SQL Injection |
| `SELECT *` em query de produção | Quebra silenciosa quando schema muda |
| Editar um script de migration já aplicado em vez de criar um novo | Corrompe o histórico de schema entre ambientes |
| Dois type maps diferentes (um global, um "especial" dentro de um módulo) | Comportamento de mapeamento inconsistente e imprevisível entre módulos |

## 11. Enforcement

- Code review bloqueia qualquer query cross-schema e qualquer SQL concatenado sem parametrização.
- Migration aplicada em ambiente compartilhado (staging/produção) é imutável — reforçado por convenção de nomenclatura sequencial e, quando o pipeline de CI for definido, por checagem automática de hash dos scripts já aplicados (comportamento nativo do DbUp).
