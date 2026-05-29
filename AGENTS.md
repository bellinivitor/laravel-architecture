# AGENTS.md — Guia de Arquitetura Back-end Laravel para Agentes de IA

> Versão: 2.0 — Laravel 12+ / PHP 8.4+
> Este arquivo é a fonte de verdade para gerar/alterar código neste projeto. Siga-o literalmente.
> Documento de referência humano completo: `README.md`.

## Regras inegociáveis (leia primeiro)

1. **Idioma do código é inglês.** Nomes de classes, métodos, variáveis, comentários → inglês. Mensagens voltadas ao usuário → via `__()` (traduzíveis).
2. **Fluxo obrigatório de uma requisição:** `Route → FormRequest (validação) → Controller (autorização + monta DTO) → Action (lógica) → Resource (resposta)`.
3. **Controller é thin.** Nunca contém regra de negócio nem acesso direto a banco.
4. **Lógica de negócio vive em Actions.** Uma Action = uma operação.
5. **A camada Domain nunca recebe HTTP.** Action recebe DTO, nunca `Request`/`FormRequest`/array cru.
6. **Sem Repository.** Acesso a dados com filtros = QueryBuilder customizado. (Justificativa na seção correspondente.)
7. **Toda escrita roda em transação** (`DB::beginTransaction`/`commit`/`rollBack`).
8. **Toda resposta de API passa por Resource.** Nunca retorne model cru.
9. **Não crie interface com implementação única.** Injete a classe concreta direto.
10. **PHP moderno:** `readonly class`, `final readonly class` (DTOs), backed enums, constructor promotion, tipos explícitos.

---

## Mapa de diretórios — onde cada coisa vai

```
app/
├── Http/
│   ├── Controllers/          # Thin. Autoriza, monta DTO, chama Action, retorna Resource.
│   ├── Requests/             # FormRequests (validação)
│   ├── Middleware/
│   └── Resources/            # (opcional aqui; preferir Domain/{D}/Resources/)
├── Models/{Domain}/          # Models Eloquent, estendem DefaultModel
│   └── Post/Post.php
├── Models/DefaultModel.php   # Model base abstrato (todo model estende)
├── Events/
└── Providers/

Domain/{Domain}/
├── Actions/                  # Lógica de negócio (readonly class + __invoke)
├── DataTransferObjects/      # DTOs (final readonly class, implementam DataTransferObjectInterface)
├── Collections/              # Eloquent Collections customizadas
├── Enums/                    # Backed enums (string|int)
├── Exceptions/               # Exceptions do domínio
├── Interfaces/               # Contratos do domínio (ex: state interfaces)
├── Jobs/                     # Background jobs (delegam lógica pesada a Actions)
├── Notifications/
├── Policies/
├── QueryBuilders/            # Queries customizadas (substituem Repository)
├── Resources/                # API Resources do domínio
└── States/                   # State Pattern (subpastas por eixo se houver vários)

Domain/Shared/
├── Interfaces/               # DataTransferObjectInterface, etc.
└── Exceptions/               # DomainHttpException, DomainException (bases)

tests/
├── Unit/                     # Actions, DTOs, lógica isolada
└── Feature/                  # Rotas HTTP ponta a ponta
```

Criar novo domínio: `php artisan make:domain {NomeDomain}`.
Criar model: `php artisan make:model`. (Enums são criados manualmente dentro do domínio — NÃO há `make:enum`.)

---

## DefaultModel (base de todos os models)

- Fica em `app/Models/DefaultModel.php`, `abstract`, estende `Model`, usa `HasFactory`.
- Concentra anotações PHPDoc de métodos estáticos de query (autocomplete da IDE).
- NUNCA coloque lógica de domínio aqui — mantenha genérico.

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

/**
 * @method static \Illuminate\Database\Eloquent\Builder query()
 * @method static static|null find(int|mixed $id, array $columns = ['*'])
 * @method static static findOrFail(int|mixed $id, array $columns = ['*'])
 * @method static static create(array $attributes)
 * @property \Carbon\Carbon $created_at
 * @property \Carbon\Carbon $updated_at
 * // ... demais @method de query conforme necessário
 */
abstract class DefaultModel extends Model
{
    use HasFactory;
}
```

---

## Models

Local: `app/Models/{Domain}/`. Estendem `DefaultModel`.

DEVE:
- Definir `$fillable` explicitamente (mass assignment).
- Definir relacionamentos Eloquent.
- Sobrescrever `newEloquentBuilder()` quando o domínio tiver QueryBuilder.
- Conter apenas accessors/mutators/casts simples.

NUNCA:
- Regra de negócio (vai para Action).
- `$guarded = []` — sempre `$fillable`.
- Crescer demais (muitos métodos além de relationships → extrair).

Cast de datas: `'datetime'` retorna `Carbon` mutável; use `'immutable_datetime'` para `CarbonImmutable`.
Cast de enum: ao persistir um backed enum (ex: status via State), declare `'status' => PostStatus::class` em `$casts`.

```php
namespace App\Models\Post;

use App\Models\DefaultModel;
use App\Models\User;
use Domain\Post\Enums\PostStatus;
use Domain\Post\QueryBuilders\PostQueryBuilder;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends DefaultModel
{
    protected $fillable = [
        'title',
        'author_id',
        'text',
    ];

    protected $casts = [
        'published_at' => 'immutable_datetime',
        'status' => PostStatus::class,
    ];

    public function newEloquentBuilder($query): PostQueryBuilder
    {
        return new PostQueryBuilder($query);
    }

    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'author_id');
    }
}
```

---

## Controllers

Local: `app/Http/Controllers/`.

DEVE (e só isso):
1. Receber `FormRequest` por injeção (validação automática).
2. Autorizar com `Gate::authorize('ability', Model::class)`.
3. Montar o DTO (`PostDTO::fromRequest($request)`).
4. Chamar a Action (injetada por type-hint, resolvida pelo container).
5. Retornar via Resource.

NUNCA:
- Regra de negócio, `Model::create()`, `DB::` direto.
- Validação inline (sempre FormRequest).
- `$this->authorize()` — a trait `AuthorizesRequests` não existe mais no Controller base. Use `Gate::authorize()` ou middleware `can` via `HasMiddleware`.

Autenticação: implemente `HasMiddleware` retornando `new Middleware('auth:sanctum')`.

```php
namespace App\Http\Controllers;

use App\Http\Requests\SearchPostRequest;
use App\Http\Requests\StorePostRequest;
use App\Models\Post\Post;
use Domain\Post\Actions\StorePostAction;
use Domain\Post\DataTransferObjects\PostDTO;
use Domain\Post\DataTransferObjects\PostSearchDTO;
use Domain\Post\Resources\PostResource;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;
use Illuminate\Support\Facades\Gate;

class PostController extends Controller implements HasMiddleware
{
    public static function middleware(): array
    {
        return [
            new Middleware('auth:sanctum'),
        ];
    }

    public function index(SearchPostRequest $request): AnonymousResourceCollection
    {
        Gate::authorize('viewAny', Post::class);

        $builder = Post::search(PostSearchDTO::fromRequest($request));
        $builder->with(['author']);
        $builder->orderBy('created_at');

        return PostResource::collection($builder->cursorPaginate());
    }

    public function store(StorePostRequest $request, StorePostAction $action): PostResource
    {
        Gate::authorize('create', Post::class);

        $post = $action(
            PostDTO::fromRequest($request),
            $request->user(),
        );

        return PostResource::make($post);
    }
}
```

---

## Actions (lógica de negócio)

Local: `Domain/{Domain}/Actions/`.

DEVE:
- Ser `readonly class` com **um único método público `__invoke()`**.
- Receber dados via **DTO** (nunca Request/array).
- Envolver escrita em transação.
- Delegar para outras Actions (injetadas no construtor) quando precisar de lógica de outro domínio.
- Ser reutilizável de Controller, Job, Command ou outra Action.

NUNCA:
- Receber `FormRequest`/`Request`/array cru.
- Virar "God Class" — uma responsabilidade por Action.

Template:

```php
namespace Domain\Post\Actions;

use App\Models\Post\Post;
use App\Models\User;
use Domain\Comment\Actions\StoreCommentAction;
use Domain\Post\DataTransferObjects\PostDTO;
use Exception;
use Illuminate\Support\Facades\DB;

readonly class StorePostAction
{
    public function __construct(
        private StoreCommentAction $storeCommentAction,
    ) {}

    public function __invoke(PostDTO $postDTO, User $user): Post
    {
        try {
            DB::beginTransaction();

            $post = new Post();
            $post->fill($postDTO->toArray());
            $post->author()->associate($user);
            $post->save();

            if ($postDTO->commentDTO) {
                ($this->storeCommentAction)($postDTO->commentDTO, $post, $user);
            }

            DB::commit();
        } catch (Exception $exception) {
            DB::rollBack();
            throw $exception;
        }

        return $post;
    }
}
```

---

## DataTransferObjects (DTOs)

Local: `Domain/{Domain}/DataTransferObjects/`. São o **contrato de entrada do Domain**.

DEVE:
- Ser `final readonly class` implementando `DataTransferObjectInterface`.
- Ter factory methods retornando `static`: `fromRequest`, `fromArray`, e opcionalmente `fromModel`.
- Não conter lógica de negócio.
- `fromRequest` e `fromArray` devem montar DTOs aninhados de forma consistente (se um trata `comment`, o outro também).

A interface compartilhada (`Domain/Shared/Interfaces/DataTransferObjectInterface.php`):

```php
namespace Domain\Shared\Interfaces;

use Illuminate\Foundation\Http\FormRequest;

interface DataTransferObjectInterface
{
    public function toArray(): array;

    public static function fromRequest(FormRequest $request): static;

    public static function fromArray(array $data): static;
}
```

`fromModel` (e outros factories específicos) NÃO entram na interface — declare só no DTO que precisar, também retornando `static`.

Template:

```php
namespace Domain\Post\DataTransferObjects;

use Domain\Shared\Interfaces\DataTransferObjectInterface;
use Illuminate\Foundation\Http\FormRequest;

final readonly class PostDTO implements DataTransferObjectInterface
{
    public function __construct(
        public string $title,
        public string $text,
        public ?int $authorId = null,
        public ?CommentDTO $commentDTO = null,
    ) {}

    public function toArray(): array
    {
        return [
            'title' => $this->title,
            'text' => $this->text,
            'author_id' => $this->authorId,
        ];
    }

    public static function fromRequest(FormRequest $request): static
    {
        return new self(
            title: $request->validated('title'),
            text: $request->validated('text'),
            authorId: $request->user()?->id,
            commentDTO: $request->validated('comment')
                ? CommentDTO::fromArray($request->validated('comment'))
                : null,
        );
    }

    public static function fromArray(array $data): static
    {
        return new self(
            title: $data['title'],
            text: $data['text'],
            authorId: $data['author_id'] ?? null,
            commentDTO: isset($data['comment'])
                ? CommentDTO::fromArray($data['comment'])
                : null,
        );
    }
}
```

Fontes que constroem o DTO: Controller (`fromRequest`), Command/Job (`fromArray`), Listener (`fromModel`), outra Action (passa o DTO).

---

## FormRequests

Local: `app/Http/Requests/`.

- Use em todo método de controller que aceita input.
- Defina `rules()`. `authorize()` pode retornar `true` (autorização real fica na Policy via `Gate::authorize` no controller).

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'text' => ['required', 'string'],
            'comment' => ['nullable', 'array'],
            'comment.body' => ['required_with:comment', 'string'],
        ];
    }
}
```

---

## QueryBuilders (substituem Repository)

Local: `Domain/{Domain}/QueryBuilders/`. Estendem `Illuminate\Database\Eloquent\Builder`.

- Model habilita via `newEloquentBuilder()`. Métodos ficam chamáveis estaticamente: `Post::search($dto)`.
- Métodos de busca recebem **DTO** como parâmetro.

```php
namespace Domain\Post\QueryBuilders;

use Domain\Post\DataTransferObjects\PostSearchDTO;
use Illuminate\Database\Eloquent\Builder;

class PostQueryBuilder extends Builder
{
    public function search(PostSearchDTO $searchDTO): static
    {
        if ($searchDTO->id) {
            $this->where('id', $searchDTO->id);
        }

        if ($searchDTO->title) {
            $this->where('title', 'ilike', '%' . str_replace(' ', '%', $searchDTO->title) . '%');
        }

        if ($searchDTO->authorId) {
            $this->where('author_id', $searchDTO->authorId);
        }

        return $this;
    }
}
```

**NUNCA crie Repository.** Motivos: o Eloquent já abstrai o banco (multi-driver); Repository sobre ORM vira camada anêmica de passthrough; perde relationships/eager loading/scopes/paginação/chunk/cursor; explode em métodos (`findByX`). O QueryBuilder entrega queries reutilizáveis/encapsuladas/testáveis estendendo o Eloquent em vez de escondê-lo.

---

## Enums

Local: `Domain/{Domain}/Enums/`. Criados manualmente (sem artisan).

- SEMPRE backed enum (`string` ou `int`).
- Para status, tipos, categorias — qualquer conjunto fixo de valores.

```php
namespace Domain\Post\Enums;

enum PostStatus: string
{
    case Draft = 'draft';
    case Published = 'published';
    case Archived = 'archived';
}
```

---

## States (State Pattern) + Enum

Local: `Domain/{Domain}/States/`. Use quando trocar de status carrega responsabilidade (side effects, validações, permissões, orquestração). Para status que é só label sem comportamento → use Enum simples validado na Action.

**Como State e Enum se compõem:** o enum é o dado persistido; o State é o comportamento que decide o próximo status. Cada State retorna o valor do enum.

Regras:
- Cada estado = classe separada implementando uma interface de estado do domínio (ex: `PostStatusState`).
- Único método público `handle()` **retorna o enum de status**.
- Estado é **puro**: não recebe o model, não toca no banco, não dispara side effects. Só responde "para qual status essa transição leva". Guarda de transição inválida (lançar exception) pode viver no `handle()`.
- Estados são `readonly class`, nomeados pelo status (`PublishedState`, `ArchivedState`).
- Vários eixos de status → subpastas com interface própria (`States/Bank/`, `States/Card/`). Um eixo só → direto em `States/`.
- A **Action recebe a interface do estado**, chama `handle()`, persiste (e side effects) na transação.

Interface:

```php
namespace Domain\Post\Interfaces;

use Domain\Post\Enums\PostStatus;

interface PostStatusState
{
    public function handle(): PostStatus;
}
```

Estado concreto:

```php
namespace Domain\Post\States;

use Domain\Post\Enums\PostStatus;
use Domain\Post\Interfaces\PostStatusState;

readonly class PublishedState implements PostStatusState
{
    public function handle(): PostStatus
    {
        return PostStatus::Published;
    }
}
```

Action que aplica a transição:

```php
namespace Domain\Post\Actions;

use App\Models\Post\Post;
use App\Models\User;
use Domain\Post\Interfaces\PostStatusState;
use Illuminate\Support\Facades\DB;
use Throwable;

readonly class ChangePostStatusAction
{
    /**
     * @throws Throwable
     */
    public function __invoke(Post $post, PostStatusState $state, User $executedBy): void
    {
        try {
            DB::beginTransaction();

            $post->setAttribute('status', $state->handle());
            $post->save();

            DB::commit();
        } catch (Throwable $throwable) {
            DB::rollBack();
            throw $throwable;
        }
    }
}
```

Ponto de entrada escolhe o estado concreto: `($action)($post, new PublishedState(), $request->user());`

---

## Resources

Local: `Domain/{Domain}/Resources/`.

- USE em todas as respostas de API.
- Defina explicitamente os campos retornados — nunca model cru.
- Controle exposição de campos sensíveis aqui.

---

## Policies / Notifications / Collections / Jobs

- **Policies** (`Domain/{D}/Policies/`): regras de autorização, acionadas por `Gate::authorize`.
- **Notifications** (`Domain/{D}/Notifications/`): email/SMS/push.
- **Collections** (`Domain/{D}/Collections/`): Eloquent Collections customizadas.
- **Jobs** (`Domain/{D}/Jobs/`): uma responsabilidade; idempotentes quando possível; logam falhas/sucessos; **lógica pesada delegada a Actions**.

---

## Comunicação entre domínios — tabela de decisão

| Situação | Mecanismo |
|---|---|
| Precisa do **retorno**, síncrono, **mesma transação** | **Injeção direta da Action** no construtor (container resolve via `__invoke`) |
| **Side effect** que o domínio de origem não precisa conhecer (notificar, indexar, analytics) | **Evento de domínio** (`PostPublished`) + listeners nos outros domínios |

Regra: precisa do valor e pertence à transação → injeta Action. "Aconteceu X, outros podem reagir" → dispara evento. Mantém domínios desacoplados e evita ciclos. Não crie interface para Action de implementação única.

---

## Exceptions e respostas de erro — tabela de decisão

| Contexto | Base a estender | Comportamento |
|---|---|---|
| Erro no **runtime da API** | `DomainHttpException` (estende `Symfony\...\HttpException`) | Laravel renderiza JSON automaticamente com o status. **Não mexer no handler.** |
| Erro **fora do HTTP** (jobs/background) | `DomainException` (estende `RuntimeException`) | Tratado pela fila (`failed_jobs`, retry/backoff). Sem render. |

Formato de erro = padrão do Laravel (não inventar envelope): validação → `{ "message", "errors": {...} }` (422); demais → `{ "message" }`. Mapeamentos automáticos: `ValidationException`→422, `AuthenticationException`→401, `AccessDeniedHttpException`→403, `ModelNotFoundException`/`NotFoundHttpException`→404. Em produção (`APP_DEBUG=false`) nada de stack trace.

Bases (em `Domain/Shared/Exceptions/`):

```php
namespace Domain\Shared\Exceptions;

use Symfony\Component\HttpKernel\Exception\HttpException;

abstract class DomainHttpException extends HttpException
{
}
```

```php
namespace Domain\Shared\Exceptions;

use RuntimeException;

abstract class DomainException extends RuntimeException
{
}
```

HTTP exception concreta (mensagem via `__()`; NÃO use `__()` em valor padrão de parâmetro — resolva no corpo):

```php
namespace Domain\Post\Exceptions;

use Domain\Shared\Exceptions\DomainHttpException;
use Symfony\Component\HttpFoundation\Response;

class PostNotFoundException extends DomainHttpException
{
    public function __construct(?string $message = null)
    {
        parent::__construct(
            Response::HTTP_NOT_FOUND,
            $message ?? __('post.not_found'),
        );
    }
}
```

Uso na API: `$post = Post::find($id) ?? throw new PostNotFoundException();` → 404.

Domain exception em job:

```php
namespace Domain\Post\Exceptions;

use Domain\Shared\Exceptions\DomainException;

class PostProcessingFailedException extends DomainException
{
}
```

```php
public function handle(): void
{
    $post = Post::find($this->postId)
        ?? throw new PostProcessingFailedException(
            __('post.processing.not_found', ['id' => $this->postId]),
        );

    // ... processamento ...
}
```

---

## Testes (Pest PHP)

- **Unit** (`tests/Unit/`): Actions, DTOs, lógica isolada.
- **Feature** (`tests/Feature/`): rotas HTTP ponta a ponta + estado do banco.
- Padrão **Arrange → Act → Assert** em todo teste.
- Factories espelham uso real. `->group('Tipo', 'Dominio')`. Datasets via `->with()`.

Unit:

```php
// tests/Unit/Post/StorePostActionTest.php
it('stores a post successfully', function (User $admin) {
    // Arrange
    $postDTO = PostDTO::fromArray(Post::factory()->raw());

    // Act
    $result = app(StorePostAction::class)($postDTO, $admin);

    // Assert
    expect($result)->toBeInstanceOf(Post::class);
    expect($result->title)->toBe($postDTO->title);
})->with('admin')->group('Unit', 'Post');
```

Feature:

```php
// tests/Feature/Post/PostStoreTest.php
test('admin can store a post via API', function (User $admin) {
    // Arrange
    $postData = Post::factory()->raw();

    // Act
    $response = actingAs($admin)->postJson(route('posts.store'), $postData);

    // Assert
    $response->assertCreated();
    $this->assertDatabaseHas('posts', ['title' => $postData['title']]);
})->with('admin')->group('Feature', 'Post');
```

---

## Receita: adicionar um endpoint de escrita (passo a passo para o agente)

1. **Model** em `app/Models/{Domain}/` (estende `DefaultModel`, `$fillable`, casts, relationships) se ainda não existir.
2. **FormRequest** em `app/Http/Requests/` com `rules()`.
3. **DTO** em `Domain/{Domain}/DataTransferObjects/` (`final readonly class`, `fromRequest`/`fromArray`).
4. **Action** em `Domain/{Domain}/Actions/` (`readonly class`, `__invoke`, transação, recebe DTO).
5. **Policy** + ability registrada; chamar `Gate::authorize` no controller.
6. **Resource** em `Domain/{Domain}/Resources/` com campos explícitos.
7. **Controller** thin: autoriza → monta DTO → chama Action → retorna Resource.
8. **Rota** apontando para o controller (nome de rota usado nos testes, ex: `posts.store`).
9. **Exceptions** de domínio conforme contexto (`DomainHttpException` na API, `DomainException` em job).
10. **Testes** Unit (Action) + Feature (rota), padrão AAA.

## Checklist de revisão antes de concluir

- [ ] Controller sem regra de negócio nem acesso a banco.
- [ ] Action é `readonly class` com só `__invoke`, recebe DTO, escreve em transação.
- [ ] DTO é `final readonly class`, implementa a interface, factories retornam `static`.
- [ ] Nenhum Repository criado; queries com filtro no QueryBuilder.
- [ ] Resposta via Resource; sem model cru.
- [ ] Autorização via `Gate::authorize` (não `$this->authorize()`); auth via `HasMiddleware`.
- [ ] Backed enums para conjuntos fixos; cast de enum no model quando persistido.
- [ ] Exception certa para o contexto (HTTP vs domain); mensagens via `__()`.
- [ ] Sem interface de implementação única.
- [ ] Código em inglês; PSR/PER (4 espaços, sem tabs, vírgula final em multilinha).
- [ ] Testes Unit + Feature no padrão AAA, com `->group()`.
