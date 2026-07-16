# RULES — Modules / Shared

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` (seção 8) e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui. Fecha as definições que os demais `RULES.md` de `Modules` já assumiram como existentes (`Result<T>`, `Error`, `Entity`, `AggregateRoot`, `ToActionResult()`, `IModuleInstaller`).

## 1. Missão

`Shared` é código cruzado entre módulos — mas, ao contrário de `Contracts`,
não pertence a nenhum módulo específico. Vive em `Modules/Shared/`, no mesmo
nível das pastas de módulo, mas não é um módulo (`03-MODULES/RULES.md`
seção 2): sem `Contracts`, sem schema de banco, sem `ModuleInstaller.cs`
próprio.

Regra de dependência: **`Shared` nunca depende de nenhum módulo específico**
(dependência é sempre módulo → `Shared`, nunca o contrário). `Shared` pode
depender de `Infrastructure` quando necessário (ex: `Web` conhece
`Result`/`Error` mas não precisa de `Infrastructure`), seguindo a mesma
direção permitida para qualquer código de módulo.

## 2. Estrutura de pastas

```
Modules/
└── Shared/
    ├── IModuleInstaller.cs           # contrato de descoberta — ver seção 4
    ├── Contracts/                    # I<NomeModulo> + IntegrationEvents de TODOS os módulos — ver CONTRACTS/RULES.md
    │   ├── I<NomeModulo>.cs
    │   └── IntegrationEvents/
    │       └── <EventoNoPassado>Event.cs
    ├── Helpers/
    ├── Kernel/
    │   ├── Entity.cs
    │   ├── AggregateRoot.cs
    │   ├── Result.cs
    │   ├── Error.cs
    │   └── DomainException.cs
    ├── Security/
    │   └── GruposDeAcesso.cs          # nomes de policy de autorização — ver 02-INFRASTRUCTURE/SECURITY/RULES.md seção 7
    └── Web/
        └── ResultExtensions.cs        # ToActionResult() — usado pelo Controller
```

`Contracts/` aqui é diferente de `Modules/<NomeModulo>/Contracts/` (que só
tem `Dtos/`) — é a pasta que reúne a superfície pública **de todos os
módulos juntos**, o que é exatamente o motivo dela morar em `Shared` e não
dentro de cada módulo (`CONTRACTS/RULES.md` seção 1). Cada arquivo dentro
dela ainda "pertence" logicamente a um módulo publicador — a organização
física comum não apaga essa propriedade, só resolve o problema de
compilação (nenhum módulo referencia outro diretamente).

## 3. `Kernel` — os primitivos que todo módulo herda

### `Entity` / `AggregateRoot`

```csharp
public abstract class Entity
{
    public Guid Id { get; protected set; }

    public override bool Equals(object? obj) =>
        obj is Entity other && other.GetType() == GetType() && other.Id == Id;

    public override int GetHashCode() => Id.GetHashCode();
}

public abstract class AggregateRoot : Entity
{
}
```

Toda `Entity` de todo módulo herda de uma dessas duas — ver `ENTITIES/RULES.md`.
Igualdade é sempre por identidade (`Id`), nunca por valor de todas as
propriedades.

### `Result<T>` / `Result`

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public Error? Error { get; }

    private Result(bool isSuccess, T? value, Error? error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(Error error) => new(false, default, error);
}

public class Result : Result<Unit> // operações sem valor de retorno
{
    public static Result Success() => (Result)Result<Unit>.Success(Unit.Value);
    public static new Result Failure(Error error) => (Result)Result<Unit>.Failure(error);
}
```

Todo `Handler` retorna `Result<TDto>` (ou `Result` para operações sem valor
de retorno) — nunca lança exceção para representar uma falha de negócio
esperada (`HANDLER/RULES.md` seção 6).

### `Error` / `ErrorType`

```csharp
public enum ErrorType { Validation, NotFound, Conflict, Unauthorized, Unexpected }

public record Error(string Code, string Message, ErrorType Type)
{
    public static Error Validation(string message, string code = "validation_error") => new(code, message, ErrorType.Validation);
    public static Error NotFound(string message, string code = "not_found") => new(code, message, ErrorType.NotFound);
    public static Error Conflict(string message, string code = "conflict") => new(code, message, ErrorType.Conflict);
    public static Error Unauthorized(string message, string code = "unauthorized") => new(code, message, ErrorType.Unauthorized);
}
```

Os métodos estáticos (`Error.Validation(...)`, `Error.NotFound(...)`) são a
forma padrão de criar um `Error` — evita espalhar `new Error(...)` com
`ErrorType` errado por engano em algum `Handler`.

### `DomainException`

```csharp
public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}
```

Lançada por uma `Entity` quando um invariante é violado (`ENTITIES/RULES.md`
seção 3) — representa um bug (estado impossível tentado), não uma falha de
negócio esperada.

## 4. `IModuleInstaller` — o contrato de descoberta

```csharp
public interface IModuleInstaller
{
    IServiceCollection Install(IServiceCollection services, IConfiguration configuration);

    void Use(IApplicationBuilder app) { } // default method — só sobrescrito por módulo que precisa tocar no pipeline HTTP
}
```

Vive na raiz de `Shared` (não em `Kernel` — não é um primitivo de domínio; não
em `Web` — não é específico de HTTP) porque é um contrato de composição que
tanto o Host quanto todo módulo precisam enxergar. Cada módulo implementa
essa interface numa classe `<NomeModulo>ModuleInstaller` (`03-MODULES/RULES.md`
seção 6); o Host descobre essas classes via assembly scanning e chama
`Install()`/`Use()` automaticamente (`01-HOST/RULES.md` seção 3) — nenhum
módulo novo exige alterar código do Host.

`Use(IApplicationBuilder)` é um **default interface method** (C# 8+) — a
imensa maioria dos módulos não precisa de nada no pipeline HTTP além de
`MapControllers()` (já cuidado pelo Host globalmente), então não sobrescrever
é o caso comum, sem forçar toda implementação a escrever um corpo vazio.

## 5. `Web` — `ResultExtensions.ToActionResult()`

```csharp
public static class ResultExtensions
{
    public static IActionResult ToActionResult<T>(this Result<T> result, int successStatusCode = StatusCodes.Status200OK)
    {
        if (result.IsSuccess)
            return new ObjectResult(result.Value) { StatusCode = successStatusCode };

        return result.Error!.Type switch
        {
            ErrorType.Validation => new BadRequestObjectResult(result.Error),
            ErrorType.NotFound => new NotFoundObjectResult(result.Error),
            ErrorType.Conflict => new ConflictObjectResult(result.Error),
            ErrorType.Unauthorized => new ObjectResult(result.Error) { StatusCode = StatusCodes.Status401Unauthorized },
            _ => new ObjectResult(result.Error) { StatusCode = StatusCodes.Status500InternalServerError }
        };
    }
}
```

Essa é a implementação única da tabela definida em `CONTROLLER/RULES.md`
seção 5 — nenhum `Controller` reimplementa esse `switch` localmente.

## 5.1 `Security` — nomes de grupo de acesso

```csharp
// Modules/Shared/Security/GruposDeAcesso.cs
namespace Shared.Security;

public static class GruposDeAcesso
{
    public const string Administrador = "Administrador";
    public const string UsuarioPadrao = "UsuarioPadrao";
}
```

Única fonte de nomes de policy de autorização usados em
`[Authorize(Policy = ...)]` pelos Controllers de qualquer módulo — evita
string literal duplicada/divergente (`02-INFRASTRUCTURE/SECURITY/RULES.md`
seção 7 tem o detalhamento completo). `GlobalExceptionMiddleware` e a
hierarquia `AppException`, por outro lado, **não** vivem aqui — moram em
`Infrastructure/Security/Exceptions/` (`SECURITY/RULES.md` seção 6), porque
dependem de tipo do ASP.NET Core (`RequestDelegate`, `HttpContext`) do
mesmo jeito que `RabbitMqConsumerBase` mora em `Infrastructure/Messaging`
e não em `Shared` (`MESSAGING/RULES.md` seção 4 tem raciocínio análogo).

## 6. `Helpers` — utilitário técnico, sem estado, sem regra de negócio

Extensões e formatadores genéricos que não se encaixam em `Kernel` (que é
especificamente sobre modelagem de domínio/aplicação) nem em `Web`
(especificamente sobre HTTP). Exemplo: extensão de formatação de string,
helper de data/hora. Se um "helper" começa a conter uma regra que só faz
sentido para um módulo específico, ele não pertence aqui — volta para dentro
do módulo.

## 7. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| Tipo em `Kernel` que só faz sentido para um módulo específico | Deveria estar dentro do módulo, não em código compartilhado (`ARCHITECTURE-RULES.md` seção 8) |
| `Helper` contendo regra de negócio | `Helpers` é utilitário técnico puro, sem conhecimento de domínio |
| `Shared` referenciando qualquer `Modules/<NomeModulo>` específico | Inverteria a única direção de dependência permitida (módulo → Shared) |
| Reimplementação local de `ToActionResult()` dentro de um `Controller` | Diverge da única fonte de verdade definida aqui |
| `Result<T>` reimplementado localmente dentro de um módulo | Duplica um primitivo que já existe — todo módulo usa o mesmo tipo |
| Uma segunda interface de descoberta de módulo (ex: `IModuleBootstrapper` paralelo) | `IModuleInstaller` é o único contrato — o Host só escaneia por ele |

## 8. Enforcement

- Code review bloqueia qualquer `using Modules.<NomeModulo>` dentro de `Modules/Shared/`.
- Code review confere que todo `Handler` usa o `Result<T>`/`Error` daqui, nunca uma variação local.
- Code review confere que todo `<NomeModulo>ModuleInstaller` é `public` com construtor parameterless (`01-HOST/RULES.md` seção 3).
