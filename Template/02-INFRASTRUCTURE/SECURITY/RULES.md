# RULES — Infrastructure / Security

> Herda todas as regras de [`00-PRINCIPLES/ARCHITECTURE-RULES.md`](../../00-PRINCIPLES/ARCHITECTURE-RULES.md). Este documento especializa, nunca substitui.

## 1. Missão

Esta camada fornece o **mecanismo técnico** de autenticação (JWT) e o
**middleware genérico de tratamento de exceção** — assim como `Database`/
`Messaging`/`Cache`, ela não sabe o que é um "Pedido" nem quem é um usuário
específico, só sabe emitir/validar token e transformar uma exceção em
resposta HTTP. **Quem é o usuário, qual senha ele tem, e a quais grupos ele
pertence é dado de negócio — mora num módulo (seção 5), nunca aqui.**

## 2. Decisões técnicas desta camada

| Decisão | Escolha |
|---|---|
| Mecanismo de autenticação | **JWT** (Bearer token), `Microsoft.AspNetCore.Authentication.JwtBearer` |
| Autorização | **Baseada em Policy** (`[Authorize(Policy = "...")]`), nunca `[Authorize(Roles = "...")]` cru espalhado pelos Controllers |
| Hash de senha | **`IPasswordHasher`** — `Rfc2898DeriveBytes`/PBKDF2 (biblioteca padrão do .NET, sem dependência externa) |
| Exceções de negócio esperadas | Já cobertas por `Result<T>.Failure` (`HANDLER/RULES.md` seção 7) — Security só endereça exceção **não esperada** |
| Middleware de exceção | Global, único, no Host — nenhum módulo captura exceção de infraestrutura manualmente |

## 3. Estrutura de pastas

```
Infrastructure/
└── Security/
    ├── Interface/
    │   ├── ITokenService.cs         # gera e valida o JWT
    │   └── IPasswordHasher.cs       # hash/verificação de senha
    ├── Implementation/
    │   ├── JwtTokenService.cs
    │   └── Pbkdf2PasswordHasher.cs
    ├── Extensions/
    │   └── SecurityExtensions.cs    # AddSecurity(IServiceCollection, IConfiguration) — JWT bearer + policies
    ├── Options/
    │   └── JwtOptions.cs            # chave secreta, emissor, audiência, tempo de expiração (bind de appsettings)
    └── Exceptions/
        ├── AppException.cs          # base de toda exceção não-domínio tratada pelo middleware (seção 6)
        ├── NotFoundException.cs
        ├── UnauthorizedAppException.cs
        └── GlobalExceptionMiddleware.cs
```

Mesma subpasta por tipo (`Interface/`, `Implementation/`, `Extensions/`) que
`Database`/`Cache`/`Messaging` (`DATABASE/RULES.md` seção 3.1) — `Options/`
e `Exceptions/` entram como categorias próprias porque cada uma tem 2+
arquivos e um propósito coeso.

## 4. `ITokenService` — emissão e validação de JWT

```csharp
public interface ITokenService
{
    string GerarToken(Guid usuarioId, string email, IReadOnlyCollection<string> grupos);
}
```

- Recebe só o que precisa para montar as claims — nunca recebe a `Entity`
  de usuário inteira (o módulo dono do usuário, seção 5, é quem decide o
  que vira claim, não `Infrastructure/Security`).
- Claims mínimas no token: `sub` (usuarioId), `email`, e uma claim de grupo
  repetida por grupo (`ClaimTypes.Role` por convenção do
  `JwtBearer`/`[Authorize(Roles=...)]`, ou uma claim customizada `"grupo"`
  se a política de autorização usar `Policy` em vez de `Roles` — ver
  seção 7).
- Tempo de expiração, chave de assinatura (`SymmetricSecurityKey`), emissor
  e audiência vêm de `JwtOptions` (seção 3), lidos de
  `Infrastructure:Security:Jwt:*` no `appsettings` — a chave secreta em si
  **nunca** fica versionada em `appsettings.json` (mesma regra de
  `01-HOST/RULES.md` seção 4 para qualquer segredo), só em
  `appsettings.{Environment}.json` fora do controle de versão, variável de
  ambiente, ou secret manager.
- `ITokenService` não sabe validar credencial (usuário+senha) — só sabe
  transformar "isso já foi validado, aqui estão os dados" em um token
  assinado. Validação de credencial é responsabilidade do módulo dono do
  usuário (seção 5).

## 5. Quem é o usuário — módulo `Auth` (ou `Identity`), não `Infrastructure`

**Decisão:** autenticação/autorização segue exatamente o mesmo padrão de
qualquer outro domínio desta arquitetura — um módulo de negócio
(`03-MODULES/RULES.md`), com schema próprio (`auth`), `Entity` própria
(`Usuario`), `Repository`, `Handler`s (`LoginHandler`,
`RegistrarUsuarioHandler`), `Controller` (`POST /api/auth/login`) e
`Contracts`. A diferença para um módulo "de negócio" comum: ele consome
`ITokenService`/`IPasswordHasher` de `Infrastructure/Security` (do mesmo
jeito que qualquer módulo consome `IEventBus`/`ICacheService`).

```
Modules/Auth/
├── Auth.csproj                       # referencia só Infrastructure + Shared, igual a qualquer módulo
├── Entities/
│   └── Usuario.cs                    # Id, Email, SenhaHash, Grupos (IReadOnlyCollection<string>)
├── Repositories/
│   ├── IUsuarioRepository.cs         # public
│   └── UsuarioRepository.cs          # internal
├── Handler/
│   ├── LoginHandler.cs               # valida credencial (IPasswordHasher) + gera token (ITokenService)
│   └── RegistrarUsuarioHandler.cs
├── Commands/
│   ├── LoginCommand.cs
│   └── RegistrarUsuarioCommand.cs
├── Contracts/Dtos/
│   └── TokenDto.cs                   # { Token, ExpiraEm }
└── Controller/
    └── AuthController.cs             # POST /api/auth/login, POST /api/auth/registrar — sem [Authorize], é a porta de entrada
```

- `Usuario` **não é a mesma `Entity`** que `Pessoa` (módulo de domínio da
  aplicação, ex: `Pessoas` no AnimeList) — são conceitos diferentes:
  `Usuario` é "quem loga", `Pessoa` é "sobre quem o sistema fala". Uma
  aplicação real normalmente tem os dois e os relaciona por um Id (ex:
  `Usuario.PessoaId`), nunca funde um dentro do outro — misturar as duas
  responsabilidades numa única `Entity` acopla regra de autenticação a
  regra de domínio de negócio.
- Se a aplicação for pequena o suficiente para genuinamente não precisar
  dessa distinção (ex: `Usuario` e `Pessoa` sempre 1:1 e nunca vão divergir),
  a decisão de fundir os dois é local ao projeto, documentada explicitamente
  — nunca o padrão default desta arquitetura.
- `LoginHandler` segue a mesma forma de qualquer `Handler`
  (`HANDLER/RULES.md`): recebe `LoginCommand`, retorna
  `Result<TokenDto>.Failure(Error.Unauthorized(...))` em credencial
  inválida — nunca lança exceção para esse caso (é falha de negócio
  esperada, seção 6 abaixo não se aplica aqui).

## 6. Exceptions — hierarquia e middleware global

```csharp
// Infrastructure/Security/Exceptions/AppException.cs
public abstract class AppException : Exception
{
    protected AppException(string message) : base(message) { }
    public abstract int StatusCode { get; }
}

public class NotFoundException : AppException
{
    public NotFoundException(string message) : base(message) { }
    public override int StatusCode => StatusCodes.Status404NotFound;
}

public class UnauthorizedAppException : AppException
{
    public UnauthorizedAppException(string message) : base(message) { }
    public override int StatusCode => StatusCodes.Status401Unauthorized;
}
```

- **Isso não substitui `Result<T>.Failure`** (`HANDLER/RULES.md` seção 7) —
  continua sendo o caminho padrão para toda falha de negócio esperada
  dentro de um `Handler`. `AppException` (e subclasses) é para os poucos
  casos que acontecem **fora** de um `Handler` — ex: middleware de
  autorização rejeitando a requisição antes de qualquer `Handler` rodar,
  ou uma validação de infraestrutura (token expirado, recurso de
  configuração ausente) que não se encaixa no fluxo de `Command`/`Query`.
- `DomainException` (`SHARED/RULES.md` seção 3) continua separada e
  **não** herda de `AppException` — ela representa violação de invariante
  (bug), sempre `500`, nunca um tipo de erro esperado com status HTTP
  próprio. `AppException` representa uma condição HTTP conhecida fora do
  fluxo de `Handler`.

```csharp
// Infrastructure/Security/Exceptions/GlobalExceptionMiddleware.cs
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;

    public GlobalExceptionMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (AppException ex)
        {
            context.Response.StatusCode = ex.StatusCode;
            await context.Response.WriteAsJsonAsync(new { error = ex.Message });
        }
        catch (Exception ex)
        {
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new { error = "Erro interno inesperado." });
            // logging estruturado do ex completo — a definir com a estratégia de observabilidade do projeto
        }
    }
}
```

- Registrado no Host **antes** de autenticação/autorização no pipeline
  (seção 8) — precisa capturar exceção de qualquer estágio seguinte,
  incluindo falha de autorização se ela lançar em vez de retornar `403`
  automaticamente.
- `DomainException` cai no `catch (Exception ex)` genérico (`500`) — ela
  nunca deveria vazar até aqui na prática (violação de invariante deveria
  ser evitada pelo `Handler` antes de chamar o método da `Entity` quando é
  uma condição esperada — `ENTITIES/RULES.md` seção 3), mas se vazar, o
  tratamento correto é `500`, não um `4xx` mascarando um bug como se fosse
  entrada inválida do cliente.

## 7. Grupos de acesso por Controller — Policy-based authorization

**Decisão:** cada `Controller` (ou ação individual, quando o `Controller`
tem ações com exigências diferentes) declara o grupo exigido via
`[Authorize(Policy = "<NomeDoGrupo>")]`. As policies em si são registradas
uma única vez no Host; o nome de cada policy é uma constante compartilhada
em `Shared/Security/` (não em cada módulo, para evitar duas strings
diferentes acidentalmente apontando pro "mesmo" grupo):

```csharp
// Modules/Shared/Security/GruposDeAcesso.cs
namespace Shared.Security;

public static class GruposDeAcesso
{
    public const string Administrador = "Administrador";
    public const string UsuarioPadrao = "UsuarioPadrao";
}
```

```csharp
// Host/DependencyInjection/ModuleRegistration.cs — NUNCA dentro de
// Infrastructure/Security/Extensions/SecurityExtensions.cs (ver nota abaixo)
private static IServiceCollection AddAuthorizationPolicies(this IServiceCollection services)
{
    services.AddAuthorization(options =>
    {
        options.AddPolicy(GruposDeAcesso.Administrador, p => p.RequireRole(GruposDeAcesso.Administrador));
        options.AddPolicy(GruposDeAcesso.UsuarioPadrao, p => p.RequireRole(GruposDeAcesso.UsuarioPadrao, GruposDeAcesso.Administrador));
    });

    return services;
}
```

**Por que isso não pode viver dentro de `AddSecurity()` (Infrastructure):**
`AddSecurity()` precisaria referenciar `Shared.Security.GruposDeAcesso` para
montar as policies — mas `Infrastructure` nunca pode depender de
`Modules/Shared` (`ARCHITECTURE-RULES.md` seção 2: a única direção permitida
é `Modules → Infrastructure`). `AddSecurity()` (em `Infrastructure/Security/
Extensions/`) fica responsável só pelo mecanismo genérico (JWT bearer,
`ITokenService`/`IPasswordHasher`); o registro das *policies* com nome
concreto de grupo é chamado separadamente dentro de `AddInfrastructure()`
no Host (`Host/DependencyInjection/ModuleRegistration.cs`), que já
referencia `Shared` livremente. Esta é a decisão final — não uma das
opções em aberto.

```csharp
[ApiController]
[Route("api/catalogo/animes")]
[Authorize(Policy = GruposDeAcesso.UsuarioPadrao)]
public class AnimesController : ControllerBase { /* ... */ }
```

- `Shared/Security/GruposDeAcesso.cs` é a **única** fonte de nomes de
  grupo — nenhum Controller escreve a string literal `"Administrador"`
  diretamente, sempre a constante (evita erro de digitação silencioso que
  faria uma policy nunca casar com nenhum usuário real).
  `Shared` pode conter isso pelo mesmo motivo que contém `Contracts`
  (`ARCHITECTURE-RULES.md` seção 8): é algo que todo módulo referencia,
  não pertence a um módulo específico.
- Endpoints publicamente acessíveis sem token (ex: `POST /api/auth/login`)
  usam `[AllowAnonymous]` explicitamente — nunca a ausência do atributo
  `[Authorize]` como forma implícita de "público"; ver seção 9
  (anti-padrão).
- Se a exigência de grupo variar por ação dentro do mesmo `Controller`
  (ex: `GET` público mas `POST` só para `Administrador`), o atributo
  `[Authorize]`/`[AllowAnonymous]` vai na ação específica, não na classe.

## 8. Wiring no Host — ordem do pipeline

Atualiza a ordem padrão definida em `01-HOST/RULES.md` seção 5:

```csharp
builder.Services.AddInfrastructure(builder.Configuration);   // inclui AddSecurity() (JWT bearer) + AddAuthorizationPolicies() (seção 7)

var app = builder.Build();

app.UseMiddleware<GlobalExceptionMiddleware>();          // 1. captura qualquer exceção dali pra frente
app.UseAuthentication();                                 // 2. resolve quem é o usuário (token → claims)
app.UseAuthorization();                                   // 3. decide se o usuário pode acessar a rota
app.UseModules();                                          // 4. Use() de cada IModuleInstaller
app.MapControllers();
app.Run();
```

- `AddSecurity()` é chamado dentro de `AddInfrastructure()`
  (`01-HOST/RULES.md` seção 3), junto de `AddDatabase`/`AddMessaging`/
  `AddCache` — mesma peça fixa e conhecida, não uma lista dinâmica de
  módulo.
- `UseAuthentication()` sempre antes de `UseAuthorization()` — ordem
  exigida pelo ASP.NET Core (autorização depende do resultado da
  autenticação).

## 9. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| `Usuario`/senha/grupo modelado dentro de `Infrastructure/Security` | `Infrastructure` não conhece dado de negócio — isso é módulo (seção 5) |
| `[Authorize(Roles = "Administrador")]` com string literal repetida em Controllers diferentes | Usa a constante de `Shared/Security/GruposDeAcesso` — evita divergência silenciosa de nome de grupo |
| Controller sem `[Authorize]` nem `[AllowAnonymous]` | Ambíguo se é proposital ou esquecimento — todo Controller declara a intenção explicitamente |
| `Handler` capturando exceção de infraestrutura pra virar resposta HTTP manualmente | É o que `GlobalExceptionMiddleware` já faz — duplicação, ver `HANDLER/RULES.md` seção 7 |
| Chave secreta do JWT versionada em `appsettings.json` | Mesma regra de qualquer segredo (`01-HOST/RULES.md` seção 4) |
| `DomainException` tratada como `401`/`403`/`404` no middleware | `DomainException` é sempre bug (`500`) — só `AppException` (seção 6) tem status HTTP próprio |

## 10. Enforcement

- Code review bloqueia qualquer `Controller` sem `[Authorize]`/`[AllowAnonymous]` explícito.
- Code review bloqueia string literal de nome de grupo fora de `Shared/Security/GruposDeAcesso`.
- Code review confere que `Usuario` (módulo `Auth`) e a `Entity` de domínio principal do projeto (ex: `Pessoa`) continuam entidades separadas, a menos que a fusão tenha sido decisão explícita e documentada.
