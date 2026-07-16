# RULES — Host

> Herda todas as regras de [`00-PRINCIPLES/ARCHITECTURE-RULES.md`](../00-PRINCIPLES/ARCHITECTURE-RULES.md). Este documento especializa, nunca substitui.

## 1. Missão

O Host é o **composition root** do sistema: o único lugar onde `Infrastructure`
e todos os `Modules` são conhecidos ao mesmo tempo e conectados. Ele não
resolve problema de negócio — ele monta o ecossistema onde os módulos rodam.

Teste rápido para saber se algo pertence ao Host: *"isto decide COMO o sistema
liga, ou decide O QUE o sistema faz?"* Se for a segunda opção, não é Host.

## 2. Estrutura de pastas

```
<raiz-da-solução>/
├── <Nome>.sln
├── Directory.Build.props            # TargetFramework, Nullable, ImplicitUsings — comum a todo .csproj da solução (seção 2.1)
└── Host/
    ├── Host.csproj
    ├── Program.cs                       # composition root — ver seção 3
    ├── appsettings.json
    ├── appsettings.{Environment}.json
    ├── DependencyInjection/
    │   └── ModuleRegistration.cs        # AddInfrastructure() + AddModules() + UseModules() — ver seção 3
    └── HealthChecks/                    # agregação/endpoint de health check, se precisar de algo além do default
```

**Uma pasta, um `.csproj`, sem aninhamento duplicado.** `Host/Host.csproj`,
nunca `Host/Host/Host.csproj` — a pasta da solução (`Host/`) já é o projeto,
não existe uma segunda pasta com o mesmo nome dentro dela só para abrigar o
`.csproj`. O mesmo vale para `Infrastructure/Infrastructure.csproj`
(`02-INFRASTRUCTURE/DATABASE/RULES.md` seção 3). Isso é diferente de
`Modules/<Nome>/<Nome>.csproj` e `Test/<Nome>/...`, onde a pasta pai (`Modules/`,
`Test/`) legitimamente contém *múltiplos* projetos — lá o segundo nível de
pasta existe pra separar um módulo do outro, não é duplicação.

Nada aqui representa domínio. Se um arquivo dentro de `Host/` começa a
acumular `if` de regra de negócio, é módulo vazando para a camada errada.

O nome do projeto é **`Host`**, sem prefixo de empresa/solução composto
(nunca `<Empresa>.Host`) — mesma convenção aplicada a `Infrastructure` e a
cada módulo (`ARCHITECTURE-RULES.md`, tabela de decisões no `README.md`).

### 2.1 `Directory.Build.props` — propriedades MSBuild comuns

Na raiz da solução (acima de `Host/`, `Infrastructure/`, `Modules/`, `Test/`),
um único `Directory.Build.props` centraliza o que, de outra forma, se
repetiria em todo `.csproj`:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

O MSBuild aplica isso automaticamente a todo projeto abaixo dessa pasta —
nenhum `.csproj` individual (Host, Infrastructure, cada módulo, cada projeto
de teste) repete `<TargetFramework>`/`<Nullable>`/`<ImplicitUsings>`. Uma
propriedade que só um projeto específico precisa (ex: um pacote NuGet extra)
continua no `.csproj` daquele projeto — `Directory.Build.props` é só para o
que é comum a todos.

## 3. Registro de módulos e infraestrutura — descoberta via assembly

Regra fixa: **`Program.cs` só chama métodos de extensão**, nunca implementa
lógica de registro linha a linha — e, além disso, **nunca lista módulo por
nome**. O Host descobre e instala cada módulo automaticamente, escaneando os
assemblies referenciados, em vez de chamar um `AddModule<Nome>()` por módulo.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddInfrastructure(builder.Configuration);   // Database + Messaging + Cache + Security
builder.Services.AddModules(builder.Configuration);          // descobre e instala todo IModuleInstaller

var app = builder.Build();

app.UseAuthorization();
app.UseModules();                                             // chama Use(IApplicationBuilder) de cada IModuleInstaller
app.MapControllers();
app.Run();
```

**Sem `public partial class Program { }` no final do arquivo.** Esse trecho
só é necessário quando algum código de teste precisa referenciar o tipo
`Program` diretamente (ex: `typeof(Program)` ou `WebApplicationFactory<Program>`)
— top-level statements geram uma classe `Program` implicitamente `internal`,
então um projeto de teste em outro assembly não a enxergaria sem essa
declaração parcial pública. Como `Test/Architecture/` obtém o assembly do
Host via `Assembly.Load("Host")` (mesmo padrão usado para os módulos —
`04-TEST/CONTRACT/RULES.md` seção 3), e nenhum teste desta arquitetura usa
`WebApplicationFactory` (Integration test sobe infraestrutura real via
Testcontainers, não o pipeline HTTP inteiro — `04-TEST/INTEGRATION/RULES.md`),
não existe motivo para expor `Program`. Se um cenário futuro genuinamente
precisar de `WebApplicationFactory<Program>` (ex: um teste de Integration
end-to-end batendo em HTTP real do Host), essa é a única situação que
justifica reintroduzir a linha — decisão a registrar aqui quando ocorrer.

```csharp
// Host/DependencyInjection/ModuleRegistration.cs
public static class ModuleRegistration
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDatabase(configuration);
        services.AddMessaging(configuration);
        services.AddCache(configuration);
        services.AddSecurity(configuration);       // JWT bearer + ITokenService/IPasswordHasher (Infrastructure/Security)
        services.AddAuthorizationPolicies();        // policies por GruposDeAcesso — só pode viver aqui, não em AddSecurity() (SECURITY/RULES.md seção 7)
        return services;
    }

    public static IServiceCollection AddModules(this IServiceCollection services, IConfiguration configuration)
    {
        foreach (var installer in ObterInstallers())
            installer.Install(services, configuration);
        return services;
    }

    public static IApplicationBuilder UseModules(this IApplicationBuilder app)
    {
        foreach (var installer in ObterInstallers())
            installer.Use(app);
        return app;
    }

    private static IEnumerable<IModuleInstaller> ObterInstallers() =>
        ObterAssembliesDeProjeto()
            .SelectMany(assembly => assembly.GetTypes())
            .Where(tipo => typeof(IModuleInstaller).IsAssignableFrom(tipo) && tipo is { IsAbstract: false, IsInterface: false })
            .Select(tipo => (IModuleInstaller)Activator.CreateInstance(tipo)!);

    private static IEnumerable<Assembly> ObterAssembliesDeProjeto() =>
        DependencyContext.Default!.RuntimeLibraries
            .Where(lib => lib.Type == "project")
            .Select(lib => Assembly.Load(lib.Name));
}
```

- Cada módulo expõe **exatamente uma** classe implementando `IModuleInstaller` (interface definida em `Modules/Shared` — ver `SHARED/RULES.md`), com um método `Install(IServiceCollection, IConfiguration)` e, opcionalmente, `Use(IApplicationBuilder)` (default method vazio na interface — só sobrescrito por módulo que precisa registrar algo no pipeline HTTP, ex: um middleware local). Isso substitui o método de extensão `AddModule<Nome>()` de convenções anteriores.
- **Por que filtrar `RuntimeLibraries` por `lib.Type == "project"` e não por prefixo de nome:** a convenção de nomenclatura sem prefixo composto (seção 2) significa que módulos não compartilham um prefixo comum (`Pessoas`, `Catalogo`, `Watchlist` não têm nada em texto que os agrupe) — filtrar por nome deixou de ser viável. `lib.Type == "project"` é mais robusto: identifica qualquer `ProjectReference` da solução (incluindo `Infrastructure` e `Shared`, que não implementam `IModuleInstaller` e por isso simplesmente não aparecem no resultado do `Where` seguinte), independente de como cada um foi nomeado.
- **Por que `DependencyContext.Default.RuntimeLibraries` e não `AppDomain.CurrentDomain.GetAssemblies()`:** essa segunda API só enxerga assemblies já carregados na memória no momento da chamada. Se nada no Host referenciar um tipo de um módulo diretamente, o .NET pode nunca ter carregado aquele assembly ainda — o módulo simplesmente não apareceria no scan, **silenciosamente**, sem erro nenhum. `DependencyContext.Default.RuntimeLibraries` lê o `.deps.json` gerado no build, que lista *todo* assembly do qual o Host depende, carregado ou não — é a única fonte confiável para descoberta automática.
- Adicionar um módulo novo ao sistema **não exige tocar em `Program.cs`**: basta o módulo estar referenciado no `.csproj` do Host (`ProjectReference`) e ter uma classe `IModuleInstaller` — ele é descoberto e instalado sozinho na próxima subida.
- `Infrastructure` **não** é descoberta por assembly — continua com registro explícito (`AddInfrastructure()` chamando `AddDatabase`/`AddMessaging`/`AddCache`/`AddSecurity`), porque são peças fixas e conhecidas, não uma lista dinâmica de módulos de negócio.
- O Host **nunca** instancia `Repository`, `Handler`, `Service` ou qualquer tipo interno de módulo diretamente — mesmo quando tecnicamente possível via reflection, isso é proibido por convenção e verificado no code review. Note que boa parte desses tipos (`Repository`/`Service`/`Consumer` concretos) é `internal` (`ARCHITECTURE-RULES.md` seção 5.1) — o Host nem teria acesso em tempo de compilação, mesmo tentando.
- Registro de Controllers de cada módulo (application parts) é responsabilidade do próprio `Install()` do módulo, não do Host fazer assembly-scanning de controllers separadamente.
- Registro de health checks específicos (ex: "consumer X está processando") é responsabilidade do módulo/infra que o possui, registrado dentro do próprio `Install()`/`AddInfrastructure()`. O Host só agrega e expõe o endpoint (`app.MapHealthChecks(...)`).

## 4. Configuração (`appsettings`)

- Cada módulo lê sua própria seção namespaced, nunca uma raiz compartilhada: `Modules:Vendas`, `Modules:Estoque`. O Host não sabe (nem precisa saber) o formato interno dessas seções — ele só repassa `IConfiguration` inteiro para o `Install()` de cada `IModuleInstaller`, que extrai sua própria fatia.
- `Infrastructure` segue o mesmo princípio: seções próprias (`Infrastructure:Database`, `Infrastructure:Messaging`, `Infrastructure:Cache`), lidas internamente por `AddInfrastructure()`.
- Segredos (connection strings de produção, credenciais de broker) **nunca** ficam versionados em `appsettings.json`. Usar `appsettings.{Environment}.json` fora do controle de versão, variáveis de ambiente, ou um secret manager — a decisão de qual mecanismo específico fica registrada aqui quando escolhida (pendente).
- Feature flags globais (que afetam mais de um módulo) vivem no Host. Feature flags que só um módulo usa vivem na seção do próprio módulo.

## 5. Pipeline HTTP

- Middlewares **globais** (autenticação, autorização, tratamento de exceção não capturada, correlation id, Swagger) são configurados no Host, na ordem definida aqui, e não podem ser reordenados por módulo algum.
- Ordem padrão de pipeline (ajustar quando houver necessidade concreta, registrando o motivo):
  1. `GlobalExceptionMiddleware` (`02-INFRASTRUCTURE/SECURITY/RULES.md` seção 6)
  2. Correlation id / logging de request
  3. `UseAuthentication()` (JWT — `SECURITY/RULES.md` seção 8)
  4. `UseAuthorization()` (policies por grupo — `SECURITY/RULES.md` seção 7)
  5. `UseModules()` (`Use()` de cada `IModuleInstaller` — seção 3 acima)
  6. Roteamento para Controllers de módulo (`MapControllers()`)
- Middlewares específicos de um módulo (ex: um filtro que só faz sentido para os endpoints de Vendas) são registrados dentro do próprio módulo via `MapGroup`/filtro local — não entram na lista global do Host.

## 6. Background workers e Consumers

- Um `Consumer` (RabbitMQ) é código privado de módulo (mora em `Modules/<Nome>/Consumers/`), mas precisa ser iniciado como hosted service.
- Regra: o próprio `Install()` do módulo registra seus `Consumers` como `IHostedService`/`BackgroundService`. O Host não sabe quantos consumers um módulo tem nem o que eles consomem — só sabe que, ao descobrir e instalar o `IModuleInstaller` daquele módulo (seção 3), tudo que ele precisa rodar em background já está registrado.

## 7. Anti-padrões — o que nunca pode aparecer no Host

| Anti-padrão | Por quê é proibido |
|---|---|
| `if (pedido.Status == ...)` ou qualquer regra de domínio dentro de `Program.cs`/`Extensions/` | Regra de negócio pertence ao módulo, não ao composition root |
| Host instanciando `new AlgumRepository(...)` diretamente | Quebra a fronteira de módulo — deve passar pelo `IModuleInstaller.Install()` do módulo |
| `Program.cs` chamando `AddModuleVendas()`/listando módulo por nome | Volta a exigir editar `Program.cs` a cada módulo novo — a descoberta via assembly (seção 3) existe exatamente para eliminar isso |
| Um módulo lendo `IConfiguration` de outro módulo (ex: Vendas lendo `Modules:Estoque:...`) | Módulo só lê sua própria seção — configuração cruzada é vazamento de fronteira |
| Middleware específico de módulo registrado como middleware global do Host | Deve ficar escopado ao próprio módulo (`MapGroup`/filtro local) |
| Segredo de produção versionado em `appsettings.json` | Risco de segurança — ver seção 4 |

## 8. Enforcement

- Code review bloqueia qualquer PR que adicione lógica de negócio dentro de `Host/`.
- O teste de arquitetura definido em `ARCHITECTURE-RULES.md` (seção 5) também cobre o Host: falha o build se `Host` referenciar tipo de módulo fora de `Contracts` **ou** fora do próprio `IModuleInstaller.Install()`.
- Se um módulo referenciado no `.csproj` do Host não aparece registrado em runtime, o primeiro lugar a checar é se a classe `IModuleInstaller` dele é `public` e tem construtor parameterless — `Activator.CreateInstance` (seção 3) exige os dois.
