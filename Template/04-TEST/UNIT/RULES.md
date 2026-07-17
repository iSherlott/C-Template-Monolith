# RULES — Test / Unit

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui. Também define a estrutura geral de `Test/`, referenciada por `CONTRACT/RULES.md` e `INTEGRATION/RULES.md`.

## 1. Decisões técnicas desta camada

| Decisão | Escolha |
|---|---|
| Framework de teste | **xUnit** |
| Biblioteca de mock | **NSubstitute** |
| Escopo do Unit test | `Handler`, `Service`, `Entity` — nunca toca banco, fila ou cache reais |

## 2. Estrutura geral de `Test/`

```
Test/
├── <NomeModulo>/
│   ├── Unit/Unit.csproj              # este documento
│   ├── Contract/Contract.csproj      # ver CONTRACT/RULES.md
│   └── Integration/Integration.csproj # ver INTEGRATION/RULES.md
└── Architecture/Architecture.csproj  # exceção: não é por módulo — ver CONTRACT/RULES.md seção 3
```

Cada tipo de teste é um `.csproj` próprio, não uma pasta dentro de um único
projeto de teste. Motivo: `Unit` precisa rodar em segundos a cada commit;
`Integration` sobe containers Docker e é naturalmente mais lento — separar
por projeto permite o CI rodar cada um com a frequência/estágio adequado
(seção 6, e `INTEGRATION/RULES.md` seção 5).

**Pasta e arquivo `.csproj` sempre no mesmo padrão flat usado em
`Host`/`Infrastructure`/`Modules`** (`01-HOST/RULES.md` seção 2): a pasta
já diz o que é (`Unit/`, `Contract/`, `Integration/`), então o `.csproj`
dentro dela não repete o nome do módulo (`Unit.csproj`, nunca
`Pessoas.UnitTests/Pessoas.UnitTests.csproj`). O nome do **assembly**
continua qualificado (`<AssemblyName>Pessoas.UnitTests</AssemblyName>`
dentro do `.csproj`, seção 2.1) — só o caminho físico fica flat, não o
nome do artefato compilado.

### 2.1 Por que o assembly mantém o nome qualificado

```xml
<PropertyGroup>
  <IsPackable>false</IsPackable>
  <AssemblyName>Pessoas.UnitTests</AssemblyName>
  <RootNamespace>Pessoas.UnitTests</RootNamespace>
</PropertyGroup>
```

Duas razões práticas para declarar `AssemblyName`/`RootNamespace`
explicitamente em vez de deixar o SDK inferir do nome do arquivo
(`Unit.csproj` → seria só `Unit`):

- **`[assembly: InternalsVisibleTo("Pessoas.UnitTests")]`** (`Modules/Pessoas/AssemblyInfo.cs` — `ENTITIES/RULES.md`/`REPOSITORIES/RULES.md` sobre visibilidade `internal`) casa pelo **nome do assembly**, não pelo caminho do projeto. Se o assembly virasse só `Unit`, o mesmo nome colidiria entre `Pessoas.Unit`, `Catalogo.Unit`, etc. — múltiplos módulos declarando `InternalsVisibleTo("Unit")` apontando para "qualquer projeto chamado Unit" quebraria o isolamento que a visibilidade `internal` existe para garantir.
- **Saída do `dotnet test`/CI** mostra o nome do assembly (`Pessoas.UnitTests.dll`), não o caminho do projeto — sem o `AssemblyName` explícito, o relatório de teste mostraria `Unit.dll` repetido 4 vezes (uma por módulo), impossível de distinguir qual módulo falhou só pelo nome.

## 3. Missão do Unit test

Testa a lógica de aplicação (`Handler`), lógica reutilizável (`Service`) e
invariante de domínio (`Entity`) **isoladamente** — toda dependência externa
(`Repository`, `IEventBus`, `ICacheService`, `I<OutroModulo>`) é mockada via
NSubstitute. Nenhum teste aqui abre conexão de banco, publica em fila real,
ou fala com Redis real.

- `Controller` não é testado em Unit — é deliberadamente fino (`CONTROLLER/RULES.md`), sem lógica própria para isolar.
- `Repository` não é testado em Unit — testar Dapper contra um banco mockado não valida nada real; isso é papel do `Integration` test (`INTEGRATION/RULES.md`).

**❌ Errado — Unit test abrindo conexão real (deixou de ser Unit, virou Integration disfarçado):**

```csharp
public class PedidoHandlerTests
{
    private readonly IDbConnectionFactory _connectionFactory = new SqlServerConnectionFactory("Server=localhost;..."); // ❌ conexão real
    private readonly IPedidoRepository _repository; // ❌ implementação concreta contra banco real, não mock

    public PedidoHandlerTests() => _repository = new PedidoRepository(_connectionFactory);

    [Fact]
    public async Task Handle_DeveRetornarSucesso() // ❌ lento, frágil, depende de banco disponível pra rodar
    {
        var result = await new PedidoHandler(_repository, ...).Handle(command);
        Assert.True(result.IsSuccess);
    }
}
```

**✅ Correto — toda dependência externa mockada (exemplo completo já na seção 4):**

```csharp
private readonly IPedidoRepository _repository = Substitute.For<IPedidoRepository>(); // ✅ mock, sem I/O real
```

## 4. Exemplo — testando um `Handler`

Um `Handler` cobre vários `Command` do mesmo recurso (mutação e leitura)
(`HANDLER/RULES.md` seção 3) — a classe de teste espelha isso: **um método
de teste por cenário de cada `Handle`**, todos na mesma classe
`<Recurso>HandlerTests`, não uma classe de teste por `Command` como antes
da consolidação.

```csharp
public class PedidoHandlerTests
{
    private readonly IPedidoRepository _repository = Substitute.For<IPedidoRepository>();
    private readonly IUnitOfWorkFactory _unitOfWorkFactory = Substitute.For<IUnitOfWorkFactory>();
    private readonly IEventBus _eventBus = Substitute.For<IEventBus>();

    private PedidoHandler CriarHandler() => new(_repository, _unitOfWorkFactory, _eventBus);

    [Fact]
    public async Task Handle_CreatePedidoCommand_DeveRetornarFailureValidation_QuandoPedidoSemItens()
    {
        // Arrange
        var command = new CreatePedidoCommand(Guid.NewGuid(), Array.Empty<ItemPedidoInput>());

        // Act
        var result = await CriarHandler().Handle(command);

        // Assert
        Assert.False(result.IsSuccess);
        Assert.Equal(ErrorType.Validation, result.Error!.Type);
    }

    [Fact]
    public async Task Handle_GetPedidoByIdCommand_DeveRetornarNotFound_QuandoPedidoNaoExiste()
    {
        var idInexistente = Guid.NewGuid();
        _repository.ObterPorIdAsync(idInexistente).Returns((Pedido?)null);

        var result = await CriarHandler().Handle(new GetPedidoByIdCommand(idInexistente));

        Assert.False(result.IsSuccess);
        Assert.Equal(ErrorType.NotFound, result.Error!.Type);
    }
}
```

## 5. Convenções

- **Nomenclatura de teste:** `<Metodo>_Deve<ResultadoEsperado>_Quando<Cenario>` — descreve comportamento, não implementação.
- **Padrão AAA** (Arrange / Act / Assert) sempre explícito, inclusive com comentários curtos separando as três seções em testes menos óbvios.
- **Um comportamento por teste** — se um teste precisa de múltiplos `Assert` para validar coisas conceitualmente diferentes, provavelmente são dois testes.
- Testar cada `ErrorType` de falha esperada que o `Handler` pode retornar (`HANDLER/RULES.md` seção 7), além do caminho de sucesso.
- Invariante de `Entity` é testado direto na `Entity` (`Pedido.Criar(...)`, `Pedido.AdicionarItem(...)`), sem envolver `Handler` — mais rápido de escrever e mais preciso sobre o que está sendo validado.

## 5.1 Mockando/instanciando um tipo `internal` (`Repository`, `Service`, `Consumer`)

`Handler` (o alvo principal do Unit test) é sempre `public`, então na maioria
dos testes isso nem chega a importar — só suas dependências mockadas
(`IPedidoRepository`, interface `public`) entram no `Substitute.For<T>()`.
Mas quando o teste precisa mockar uma dependência cujo **tipo concreto** é
`internal` (raro — normalmente só a interface é mockada), ou quando o alvo do
teste é, ele mesmo, uma classe `internal` (ex: testar `VendasModuleFacade`
diretamente em vez de via `IVendasModule`), duas coisas precisam estar
presentes no módulo sendo testado (`ARCHITECTURE-RULES.md` seção 5.1):

```csharp
// Modules/<NomeModulo>/AssemblyInfo.cs
[assembly: InternalsVisibleTo("<NomeModulo>.UnitTests")]
[assembly: InternalsVisibleTo("DynamicProxyGenAssembly2")]
```

- `InternalsVisibleTo("<NomeModulo>.UnitTests")` — permite que o projeto de teste (assembly diferente) referencie o tipo `internal` diretamente em código (`new VendasModuleFacade(...)`).
- `InternalsVisibleTo("DynamicProxyGenAssembly2")` — NSubstitute usa Castle DynamicProxy internamente para gerar o proxy de mock; sem essa segunda declaração, `Substitute.For<IAlgumTipoInternal>()` ou mockar um tipo `internal` falha em runtime com erro de acesso, mesmo com o primeiro `InternalsVisibleTo` já presente.

## 6. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Unit test abrindo conexão de banco/fila/cache real | Deixa de ser Unit test — vira Integration test disfarçado, lento e frágil |
| Mock verificando detalhes de implementação irrelevantes ao comportamento (`receivedCalls.Count == 3`) | Teste frágil, quebra a cada refactor que não muda comportamento |
| Um único método de teste cobrindo múltiplos cenários com vários `Assert` não relacionados | Dificulta saber exatamente o que quebrou quando o teste falha |
| Teste de `Controller`/`Repository` em Unit | Fora do escopo desta camada — ver seção 3 |

## 7. Enforcement

- CI roda toda a suíte de `Unit` em **todo commit/PR** — não pode depender de Docker, rede, ou qualquer serviço externo para passar.
