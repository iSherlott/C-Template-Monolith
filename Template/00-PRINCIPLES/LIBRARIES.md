# LIBRARIES — Governança de dependências (NuGet)

> Herda [`ARCHITECTURE-RULES.md`](ARCHITECTURE-RULES.md). Aplica-se a **toda**
> adição de pacote NuGet, em qualquer camada (`Host`, `Infrastructure`,
> `Modules/*`, `Test/*`).

## 1. Por que este arquivo existe

Um projeto que seguia este rulebook começou a usar **MediatR** — mesmo a
decisão "sem mediator" já estando fechada em `ARCHITECTURE-RULES.md` §7 e
implementada corretamente no `AnimeList` (a implementação de referência,
`REFERENCE-IMPLEMENTATION.md`). Ninguém perguntou antes de adicionar o
pacote; o agente/dev simplesmente trouxe um hábito de outro projeto. O
resultado foi `Controller` chamando `_mediator.SendAsync(...)` em vez de
injetar o `Handler` concreto (`CONTROLLER/RULES.md` §3, §7, §8).

**Regra de ouro:** nenhum pacote NuGet novo entra na solução sem confirmação
explícita do desenvolvedor responsável pelo projeto — nem "parece óbvio",
nem "é o que eu sempre uso", nem "resolve rápido". Isso vale para qualquer
papel (`AGENTS/*.md`), incluindo o `module-agent` e principalmente o
`migration-agent` (seção 4 abaixo).

## 2. Pacotes já aprovados — fonte de verdade é o `AnimeList` real

Esta tabela reflete exatamente os `PackageReference` presentes no
`AnimeList` (implementação de referência), validado com `dotnet build` +
suíte de testes limpa. Adicionar qualquer um destes pacotes, na mesma
versão ou compatível, **não** exige nova confirmação — já é decisão
tomada. Qualquer pacote **fora** desta lista exige o processo da seção 3.

| Pacote | Versão validada | Camada | Papel |
|---|---|---|---|
| `Dapper` | `2.1.79` | `Infrastructure/Database` | Micro-ORM — acesso a dado explícito, sem mapeamento automático de escrita (`DATABASE/RULES.md`) |
| `dbup-sqlserver` | `7.2.0` | `Infrastructure/Database` | Migration runner (`DATABASE/RULES.md` §9) |
| `Microsoft.Data.SqlClient` | `7.0.2` | `Infrastructure/Database` | Driver SQL Server |
| `RabbitMQ.Client` | `7.2.1` | `Infrastructure/Messaging` | Cliente RabbitMQ, usado só dentro de `RabbitMq/` (`MESSAGING/RULES.md`) |
| `StackExchange.Redis` | `3.0.17` | `Infrastructure/Cache` | Cliente Redis, usado só dentro de `Cache/Implementation/` (`CACHE/RULES.md`) |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | `9.0.9` | `Infrastructure/Security` | Validação de JWT Bearer (`SECURITY/RULES.md`) |
| `Swashbuckle.AspNetCore` | `9.0.6` (fixado, nunca `10.x`) | `Host` | Swagger UI + botão Authorize JWT (`01-HOST/RULES.md` §3.1 — motivo da versão fixada já documentado lá) |
| `Microsoft.Extensions.DependencyModel` | `10.0.10` | `Host` | Descoberta de módulo via assembly scanning (`01-HOST/RULES.md` §3) |
| `NetArchTest.Rules` | `1.3.2` | `Test/Architecture` | Teste de arquitetura via reflection (`04-TEST/CONTRACT/RULES.md` §3) |
| `xunit` + `xunit.runner.visualstudio` | `2.9.2` / `2.8.2` | `Test/*` | Framework de teste |
| `NSubstitute` | `6.0.0` | `Test/*/Unit` | Mock (`04-TEST/UNIT/RULES.md`) |
| `Testcontainers.MsSql` | `4.13.0` | `Test/*/Integration` | SQL Server real em Docker (`04-TEST/INTEGRATION/RULES.md`) |
| `Testcontainers.RabbitMq` | `4.13.0` | `Test/*/Integration` (só nos módulos que publicam/consomem evento) | RabbitMQ real em Docker — fecha o gap de `Outbox → RabbitMQ → Consumer` (`04-TEST/INTEGRATION/RULES.md` §5, `REFERENCE-IMPLEMENTATION.md` §3) |
| `Testcontainers.Redis` | `4.13.0` | `Test/*/Integration` (só nos módulos que usam `ICacheService`) | Redis real em Docker — fecha o gap de TTL real (`04-TEST/INTEGRATION/RULES.md` §5, `REFERENCE-IMPLEMENTATION.md` §3) |
| `Microsoft.NET.Test.Sdk` + `coverlet.collector` | `17.12.0` / `6.0.2` | `Test/*` | SDK de teste + cobertura |

**Hashing de senha não usa pacote externo** (`BCrypt.Net-Next` e
similares **não** são necessários) — `Pbkdf2PasswordHasher`
(`Infrastructure/Security/Implementation/`) usa `Rfc2898DeriveBytes`, nativo
do BCL. Antes de propor uma lib de hashing nova, confirmar que o BCL
realmente não cobre o caso, porque neste projeto ele cobre.

## 3. Processo para propor um pacote fora da lista

```
1. Primeiro perguntar: "o BCL ou um pacote já aprovado (seção 2) resolve
   isso?" — a maioria dos casos resolve. Exemplos reais deste projeto:
   hashing (Rfc2898DeriveBytes, não BCrypt), roteamento Command→Handler
   (DI manual, não MediatR), mapeamento Entity→Dto (FromEntity() escrito
   à mão, não AutoMapper/Mapster), validação de forma ([Required] +
   ApiController, não FluentValidation).

2. Se genuinamente não resolve, apresentar ao dev responsável, por
   escrito, antes de tocar em qualquer .csproj:
   - Qual problema concreto o pacote resolve
   - Por que BCL/pacote já aprovado não é suficiente
   - Qual RULES.md essa adição toca ou potencialmente reabre (ex: um
     dispatcher genérico qualquer reabre ARCHITECTURE-RULES.md §7)
   - Versão proposta

3. Só adicionar ao .csproj depois de confirmação explícita ("sim, pode
   adicionar X versão Y") — não silenciosa, não implícita por o dev não
   ter reclamado de uma sugestão anterior.

4. Depois de aprovado, registrar aqui (seção 2) — pacote aprovado uma vez
   para um projeto não se propaga automaticamente para outro; cada
   projeto tem sua própria versão deste arquivo caso divirja da lista
   base do AnimeList, mas a decisão continua registrada e rastreável.
```

## 4. Cuidado redobrado — `migration-agent` (`AGENTS/MIGRATION-AGENT.md`)

O cenário de maior risco de violação silenciosa desta regra é migração:
um projeto de origem (Java Spring, .NET em outra arquitetura, etc.)
frequentemente já usa uma lib equivalente a algo que este rulebook resolve
de outro jeito — Spring Data (`Repository<T>` genérico), MapStruct/
ModelMapper (mapeamento automático), Lombok (não existe equivalente
necessário em C#), Bean Validation/`javax.validation` (`FluentValidation`
seria o equivalente C#, mas não é usado aqui). **Portar a lib junto com o
código só porque "era o que tinha lá" é exatamente o anti-padrão que gerou
este arquivo.** O `migration-agent` traduz o *padrão* (`MIGRATION-AGENT.md`
tabela de mapeamento de conceito), nunca importa a *biblioteca* de origem
sem passar pelo processo da seção 3.

## 5. Anti-padrões — o que nunca pode acontecer

| Anti-padrão | Por quê é proibido |
|---|---|
| Pacote adicionado ao `.csproj` sem confirmação explícita do dev registrada | É exatamente o incidente que originou este arquivo — reabre decisão de arquitetura silenciosamente |
| Dispatcher genérico (`MediatR`, `Mediator`, `Wolverine`, ou qualquer lib com `IRequestHandler`/`Send`) | Reabre `ARCHITECTURE-RULES.md` §7 — banido independente do nome da lib (`CONTROLLER/RULES.md` §3, §7, §8) |
| Lib de mapeamento automático (`AutoMapper`, `Mapster`) para `Entity`→`Dto` | A convenção já é `Dto.FromEntity()` escrito à mão (`CONTRACTS/RULES.md` §4) — mapeamento automático esconde quando um campo novo precisa de decisão consciente |
| Lib de ORM completo (`Entity Framework Core` como ORM de escrita, `Dapper.Contrib`) | Contradiz a decisão de SQL explícito via Dapper puro (`DATABASE/RULES.md` §8, `REPOSITORIES/RULES.md` §6.1) |
| Lib de validação declarativa (`FluentValidation`) substituindo a validação de forma na borda + invariante na `Entity` | Duplica um caminho que já existe em duas camadas já definidas (`CONTROLLER/RULES.md` §6, `ENTITIES/RULES.md` §3) |
| Trazer a lib do projeto de origem numa migração "porque já vinha com o código" | Ver seção 4 |

## 6. Enforcement

- Code review bloqueia qualquer PR que adicione `PackageReference` não listado na seção 2 sem uma confirmação explícita do dev citada na descrição do PR.
- `Test/Architecture` (`04-TEST/CONTRACT/RULES.md` §3) é o backstop automatizado para o caso mais comum (dispatcher genérico) — mas não substitui a checagem humana de code review para os demais pacotes.
