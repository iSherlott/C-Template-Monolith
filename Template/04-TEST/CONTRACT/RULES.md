# RULES — Test / Contract

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md`, `03-MODULES/RULES.md` e `03-MODULES/CONTRACTS/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

`Contract` test protege duas coisas diferentes, cobertas em dois lugares
distintos (seção 2): que a **fronteira estrutural** entre módulos não seja
violada em código, e que a **forma dos dados publicados** (`Dto`,
`IntegrationEvent`) não quebre consumidores sem aviso.

## 2. Dois tipos de contract test, dois locais diferentes

| Tipo | O que valida | Onde vive |
|---|---|---|
| **Teste de arquitetura** (solution-wide) | Nenhum módulo referencia `Entity`/`Repository`/`Handler` de outro módulo fora de `Contracts` (`ARCHITECTURE-RULES.md` seção 5) | `Test/Architecture/` — projeto único, fora da pasta por módulo, roda sobre a solução inteira |
| **Teste de forma** (por módulo) | `Dto`s e `IntegrationEvent`s publicados por este módulo mantêm compatibilidade (`CONTRACTS/RULES.md` seção 6) | `Test/<NomeModulo>/Contract/` |

Essa é a única exceção à regra "uma pasta de teste por módulo" — o teste de
arquitetura é, por natureza, sobre a relação *entre* módulos, então não faz
sentido duplicá-lo dentro de cada um.

## 3. Teste de arquitetura — `Test/Architecture/`

Usa uma biblioteca de teste de arquitetura (ex: `NetArchTest`) para validar,
via reflection sobre os assemblies compilados, as regras de
`ARCHITECTURE-RULES.md`:

```csharp
[Theory]
[MemberData(nameof(TodosOsParesDeModulo))]
public void Modulo_NaoDeveReferenciar_TiposPrivados_DeOutroModulo(string origemNome, string destinoNome)
{
    var resultado = Types.InAssembly(Assembly.Load(origemNome))
        .Should()
        .NotHaveDependencyOn(destinoNome)
        .GetResult();

    Assert.True(resultado.IsSuccessful);
}
```

- **`Assembly.Load(origemNome)` em vez de `typeof(Pedido).Assembly`:** como `Entity`/`Repository`/`Service` concretos podem ser `internal` (`ARCHITECTURE-RULES.md` seção 5.1), o projeto de teste de arquitetura nem sempre teria acesso de compilação a um `typeof(...)` desses tipos sem `InternalsVisibleTo`. Carregar o assembly pelo nome do projeto (`"Pessoas"`, `"Catalogo"`, sem prefixo — `03-MODULES/RULES.md` seção 3) evita essa dependência e mantém o teste puramente estrutural via reflection.
- `NotHaveDependencyOn(destinoNome)` é suficiente sem precisar checar `Entities`/`Repositories` separadamente: como nenhum módulo de negócio tem `ProjectReference` para outro (só para `Shared`/`Infrastructure` — `CONTRACTS/RULES.md` seção 2), qualquer dependência do assembly de origem no *nome do assembly* de destino já indica violação — o `ProjectReference` proibido simplesmente não deveria existir no `.csproj` para começo de conversa; este teste é a rede de segurança que pega a violação mesmo assim, caso alguém adicione a referência manualmente.
- Um teste parametrizado (`[Theory]`/`[MemberData]`) cobre **todo par de módulos** da solução — à medida que um módulo novo é criado, a regra já se aplica a ele automaticamente se `TodosOsParesDeModulo` itera sobre os módulos descobertos dinamicamente (mesmo mecanismo de `lib.Type == "project"` do Host — `01-HOST/RULES.md` seção 3), não uma lista de pares escrita à mão.
- Também valida `Host` (`01-HOST/RULES.md` seção 8): `Host` não pode ter dependência no assembly de nenhum módulo além de `Shared`/`Infrastructure`.

**❌ Errado — pares de módulo escritos à mão:**

```csharp
[Fact]
public void Vendas_NaoDeveReferenciar_Estoque() { /* ... */ }

[Fact]
public void Estoque_NaoDeveReferenciar_Vendas() { /* ... */ }
// ❌ um módulo novo (ex: Pagamentos) fica sem cobertura até alguém lembrar de escrever os pares dele
```

**✅ Correto — gerado a partir dos módulos descobertos dinamicamente (exemplo completo acima):**

```csharp
public static IEnumerable<object[]> TodosOsParesDeModulo() =>
    from origem in ModulosDescobertos() // mesmo mecanismo de lib.Type == "project" do Host
    from destino in ModulosDescobertos()
    where origem != destino
    select new object[] { origem, destino };
// módulo novo entra automaticamente na proxima execução, sem editar este arquivo
```

## 4. Teste de forma — `Test/<NomeModulo>/Contract/`

Valida que a forma pública (`Dto`, `IntegrationEvent`) do módulo não mudou de
forma incompatível — sem mocks, sem infraestrutura real, só reflection/
serialização pura sobre o próprio tipo:

```csharp
[Fact]
public void PedidoCriadoEvent_DeveManterFormaEsperada()
{
    var propriedades = typeof(PedidoCriadoEvent).GetProperties().Select(p => p.Name);

    Assert.Equal(
        new[] { "Id", "OccurredOn", "EventType", "PedidoId", "ClienteId", "Total" },
        propriedades);
}
```

- Se um campo é removido/renomeado sem seguir o processo de depreciação (`CONTRACTS/RULES.md` seção 6), este teste falha — é o mecanismo automático que aplica aquela regra.
- Adicionar um campo novo opcional exige atualizar a lista esperada neste teste — é uma mudança consciente, não uma quebra silenciosa.

**❌ Errado — teste de forma abrindo banco/fila real para "confirmar que funciona":**

```csharp
[Fact]
public async Task PedidoCriadoEvent_DeveManterFormaEsperada()
{
    await using var connection = new SqlConnection(connectionString); // ❌ Contract test não precisa de infra real
    var pedido = await connection.QueryFirstAsync<Pedido>(sql);
    var evento = new PedidoCriadoEvent(pedido.Id, pedido.ClienteId, pedido.Total);
    // ...
}
```

**✅ Correto — só reflection sobre o próprio tipo, sem I/O (exemplo completo acima):**

```csharp
var propriedades = typeof(PedidoCriadoEvent).GetProperties().Select(p => p.Name);
```

## 5. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Teste de arquitetura escrito manualmente par a par em vez de genérico sobre todos os módulos | Módulo novo fica sem cobertura até alguém lembrar de adicionar o teste dele |
| Teste de forma acessando banco/fila real | Foge do escopo — é só sobre a forma do tipo em código, não sobre comportamento |
| Lógica de negócio testada aqui em vez de em `Unit` | Fora do escopo desta camada |

## 6. Enforcement

- CI roda `Test/Architecture/` e todos os `Test/<NomeModulo>/Contract/` em todo commit/PR — ambos são rápidos (sem infraestrutura externa) e bloqueiam merge se falharem.
