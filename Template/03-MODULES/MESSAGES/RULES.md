# RULES — Modules / Messages

> Herda `00-PRINCIPLES/ARCHITECTURE-RULES.md` e `03-MODULES/RULES.md`. Este documento especializa, nunca substitui.

## 1. Missão

Toda mensagem voltada ao usuário final — texto de `Error.Validation(...)`,
`DomainException`, notificação — vive centralizada num único lugar por
módulo, nunca como string literal espalhada dentro de `Handler`/`Entity`.
Objetivo duplo: (1) achar/revisar todo texto do módulo num único arquivo,
sem grep pelo código inteiro; (2) preparar o terreno para idioma (`culture`)
sem precisar reescrever `Handler`/`Entity` no dia em que isso for
necessário.

## 2. `.resx` + acessador escrito à mão — não um `Dictionary<string, string>` solto, não um Designer.cs mágico

**Decisão:** apesar do nome "dicionário de mensagens" sugerir um
`Dictionary<string, string>` manual, a implementação usa **arquivos de
recurso `.resx`** (mecanismo nativo do .NET) por trás de uma classe
acessadora **escrita à mão** (`internal static class`). Motivo: o objetivo
declarado é suportar `culture` no futuro — `.resx` + `ResourceManager` já
resolve o lado difícil disso de graça, sem precisar reimplementar fallback
de idioma:

- `ResourceManager.GetString(chave, CultureInfo)` escolhe automaticamente o
  arquivo certo pela `CultureInfo` corrente (`Messages.resx` → neutro/
  fallback, `Messages.en-US.resx` → inglês, ...), sem nenhum `if`/`switch`
  de idioma escrito por nós.
- **Importante — verificado empiricamente:** rodando só `dotnet build`
  (sem Visual Studio), o `.resx` **não** gera automaticamente uma classe
  `Messages.Designer.cs` fortemente tipada — isso só acontece através do
  custom tool `ResXFileCodeGenerator` que o Visual Studio conecta na IDE.
  Um projeto compilado só via CLI/SDK (`dotnet build`/`dotnet test`) embute
  o `.resx` como recurso binário (`.resources`), mas não gera nenhum C#.
  Por isso esta arquitetura **não promete** um `Designer.cs` automático —
  a classe acessadora fortemente tipada (seção 3) é escrita à mão, uma vez
  por módulo, e chama `ResourceManager` internamente.
- Ainda assim, isso é mais barato que um `Dictionary<string, string>`
  manual: a classe acessadora tem uma propriedade de uma linha por chave
  (`public static string X => Obter(nameof(X));`), e todo o trabalho difícil
  (fallback de cultura, carregamento de satellite assembly, cache) continua
  delegado ao `ResourceManager`, não reimplementado.

## 3. Estrutura de pastas

```
Modules/<NomeModulo>/
└── Messages/
    ├── <NomeModulo>Messages.resx      # idioma neutro/fallback — dados (chave → texto)
    ├── <NomeModulo>Messages.cs        # acessador fortemente tipado, escrito à mão (ver seção 5)
    └── <NomeModulo>Messages.en-US.resx # adicionado só quando o segundo idioma for genuinamente necessário
```

- Um `.resx` (+ acessador) por módulo — nunca compartilhado entre módulos (`Messages` de um módulo é tão privado quanto `Entities`/`Handler` dele — `ARCHITECTURE-RULES.md` seção 3).
- O `.resx` neutro (sem sufixo de cultura) é criado desde o primeiro dia, mesmo em projeto monolíngue — é ele que centraliza a mensagem hoje. Os `.resx` com sufixo de cultura (`.en-US.resx`, `.es-ES.resx`, ...) só são criados quando o segundo idioma é uma necessidade real do projeto, nunca antecipados sem uso concreto (mesmo princípio de não construir abstração antes da hora — `ARCHITECTURE-RULES.md`).
- A classe acessadora é `internal` — só `Handler`/`Entity` do próprio módulo a chamam; nenhum outro módulo a referencia (nem conseguiria, é `internal`).

## 4. Convenção de chave

Chave em PascalCase, nomeada pelo **conteúdo semântico**, nunca pelo texto
literal nem pelo local de uso:

| Chave | Valor (neutro) |
|---|---|
| `PedidoSemItens` | "Pedido precisa ter ao menos um item." |
| `PedidoNaoEncontrado` | "Pedido não encontrado." |
| `PedidoJaCancelado` | "Não é possível adicionar item a um pedido que não está aberto." |
| `GeneroInvalido` | "Gênero '{0}' não existe." |

- Nunca `Erro1`, `MsgValidacao3` (não semântico) nem o texto inteiro como
  chave (`"Pedido precisa ter ao menos um item."` como chave) — uma chave
  semântica sobrevive a reformulação do texto sem precisar mudar todo lugar
  que a referencia.
- Interpolação de valor dinâmico (ex: `GeneroInvalido` acima) usa
  placeholder posicional (`{0}`) no `.resx` — nunca concatenação manual. A
  classe acessadora expõe esse caso como **método**, não propriedade
  (`GeneroInvalido(generoId)`, não `GeneroInvalido`), já encapsulando o
  `string.Format` internamente (seção 5) — quem chama nunca vê `string.Format`
  nem o índice do placeholder.

## 5. A classe acessadora — escrita à mão, uma vez por módulo

```csharp
// Modules/Pessoas/Messages/PessoasMessages.cs
using System.Globalization;
using System.Resources;

namespace Pessoas.Messages;

internal static class PessoasMessages
{
    private static readonly ResourceManager ResourceManager =
        new("Pessoas.Messages.PessoasMessages", typeof(PessoasMessages).Assembly);

    public static string NomeObrigatorio => Obter(nameof(NomeObrigatorio));
    public static string PessoaNaoEncontrada => Obter(nameof(PessoaNaoEncontrada));

    private static string Obter(string chave) =>
        ResourceManager.GetString(chave, CultureInfo.CurrentUICulture)!;
}
```

```csharp
// Caso com placeholder — Modules/Catalogo/Messages/CatalogoMessages.cs
public static string GeneroInvalido(string generoId) =>
    string.Format(CultureInfo.CurrentUICulture, Obter(nameof(GeneroInvalido)), generoId);
```

- O primeiro argumento de `new ResourceManager(...)` é o **nome lógico do
  recurso embutido**: `<RootNamespace>.<CaminhoDaPasta>.<NomeDoArquivoResx sem extensão>`,
  sempre com pontos (nunca barra) — para `Modules/Pessoas/Messages/PessoasMessages.resx`
  com `RootNamespace` `Pessoas`, o nome é `"Pessoas.Messages.PessoasMessages"`. Errar esse
  nome não é erro de compilação — é `MissingManifestResourceException` em
  runtime, na primeira chamada. Conferir com `dotnet build` + inspecionar o
  `.resources` gerado em `obj/` quando a mensagem não aparece.
- Cada propriedade repete o nome da chave via `nameof(...)` — nunca a
  string da chave duplicada manualmente (`Obter("NomeObrigatorio")`), para
  que renomear a propriedade force renomear a chave (ou vice-versa) via
  "Find usages"/rename refactor, não silenciosamente diverja.

## 6. Onde é consumido

```csharp
// Handler
if (command.Itens.Count == 0)
    return Result<PedidoDto>.Failure(Error.Validation(Messages.PedidoSemItens));

// Entity
if (Status != StatusPedido.Aberto)
    throw new DomainException(Messages.PedidoJaCancelado);
```

- `Handler` (`HANDLER/RULES.md`) e `Entity` (`ENTITIES/RULES.md`) são os
  dois consumidores — qualquer mensagem que hoje é uma string literal
  dentro de `Error.Validation(...)`/`Error.NotFound(...)`/
  `throw new DomainException(...)` migra para cá.
- `Dto`/`IntegrationEvent` (`CONTRACTS/RULES.md`) nunca contêm mensagem de
  texto solta — eles são dado estruturado; a tradução do erro pro texto
  final acontece no `Handler`, não é serializada como parte do contrato.

## 7. Cultura da requisição — resolvida pelo Host, não pelo módulo

A escolha de qual cultura usar por requisição é configurada **uma única
vez**, no Host (`RequestLocalizationMiddleware` do ASP.NET Core, lendo o
header `Accept-Language` ou uma rota/claim de preferência do usuário — a
definir junto da necessidade real do segundo idioma). Nenhum módulo decide
cultura por conta própria — a classe acessadora sempre resolve pela
`CultureInfo.CurrentUICulture` já definida pelo pipeline antes do
`Handler` rodar. Até o dia em que essa necessidade existir de fato, o Host
não precisa configurar nada além do `.resx` neutro já funcionar como
fallback padrão — não é preciso antecipar o middleware de localização sem
um segundo idioma real para justificá-lo.

## 8. Anti-padrões — o que nunca pode aparecer aqui

| Anti-padrão | Por quê é proibido |
|---|---|
| String literal de mensagem de usuário dentro de `Handler`/`Entity` (`Error.Validation("texto direto")`) | Não centralizado, impossível de traduzir depois sem caçar cada ocorrência |
| `Dictionary<string, string>` escrito à mão para mensagens | Reimplementa o que `.resx`/`ResourceManager` já resolve nativamente (seção 2) |
| Chave de mensagem não-semântica (`Msg1`, texto inteiro como chave) | Quebra ao reformular o texto; não sobrevive a manutenção |
| `.resx` compartilhado entre módulos | Viola isolamento de módulo — `Messages` é tão privado quanto `Entities` |
| Concatenação manual de valor dinâmico na mensagem (`"Gênero '" + generoId + "' não existe."`) | Usa placeholder `{0}` + método na classe acessadora (seção 5) — concatenação quebra ordem de palavra em outro idioma |
| `.resx` de cultura extra criado sem necessidade real de suportar aquele idioma | Abstração antecipada sem uso concreto |
| Classe acessadora `public` | Só `Handler`/`Entity` do próprio módulo chamam — `internal` é suficiente e reforça que não é superfície pública do módulo |

## 9. Enforcement

- Code review bloqueia string literal de mensagem de usuário passada diretamente para `Error.Validation`/`Error.NotFound`/`Error.Conflict`/`DomainException` — sempre via `<NomeModulo>Messages.<Chave>`.
- Code review confere que toda chave nova em `<NomeModulo>Messages.resx` é semântica (seção 4), não posicional/numérica, e que a classe acessadora (seção 5) tem uma propriedade/método correspondente usando `nameof`.
