# Padrão de Arquitetura Back-end Laravel

_Versão: 2.0 — Laravel 12+ / PHP 8.4+_

Este documento descreve a arquitetura do projeto desenvolvido em Laravel, especificando a função de cada camada e componente do sistema. Este modelo serve como base para futuros projetos e deve ser atualizado sempre que necessário.

**Idioma de programação: Inglês.**

### Recomendações

Antes de começar a desenvolver, é importante ter familiaridade com:

- **PHP 8.4+**: Constructor promotion, readonly classes, backed enums, intersection types, property hooks e asymmetric visibility.
- **Eloquent ORM**: Modelagem de dados, relacionamentos, casts e query scopes.
- **Routing & Middleware**: Definição de rotas, middleware stacks e interface `HasMiddleware`.
- **Autorização**: Facade `Gate`, Policies. O Controller base não inclui mais a trait `AuthorizesRequests` — utilize `Gate::authorize()` diretamente.
- **Carbon 3**: Obrigatório no Laravel 12+. Atenção: o cast `datetime` do Eloquent continua retornando `Carbon` mutável. Quando quiser imutabilidade, use o cast `immutable_datetime` (que retorna `CarbonImmutable`) e evite métodos mutantes (como `->addDay()`) sem reatribuição explícita.
- **Artisan CLI**: Geração de código (`make:model`, `make:domain`), comandos customizados.
- **Migrations & Seeders**: Versionamento de banco de dados e dados para testes.
- **Queues & Jobs**: Processamento assíncrono com Horizon.
- **Service Container & Dependency Injection**: IoC, bindings, auto-resolução.
- **Testing**: Pest PHP, factories, assertions.
- **Ecossistema Laravel**: Horizon (monitoramento de filas), Telescope (debugging), Sanctum (autenticação de API).

> _Nota: Algumas sintaxes podem variar em caso de atualização da versão do Laravel._

---

## Estrutura do Projeto

```
app/
├── Http/
│   ├── Controllers/          # Handlers HTTP (thin, delegam para Actions)
│   ├── Requests/             # Classes FormRequest de validação
│   ├── Middleware/            # Middleware HTTP
│   └── Resources/            # Transformadores de resposta API (opcional, pode ficar no Domain)
├── Models/                   # Models Eloquent organizados por domínio
│   └── Post/
│       └── Post.php
├── Events/                   # Eventos da aplicação
├── Providers/                # Service providers
Domain/
├── Post/
│   ├── Actions/              # Lógica de negócio (classes de serviço)
│   ├── DataTransferObjects/  # DTOs para transporte de dados entre camadas
│   ├── Collections/          # Collections Eloquent customizadas
│   ├── Enums/                # Enumerações do domínio
│   ├── Exceptions/           # Exceções do domínio
│   ├── Interfaces/           # Contratos e interfaces
│   ├── Jobs/                 # Jobs de background/fila
│   ├── Notifications/        # Notificações (email, SMS, push)
│   ├── Policies/             # Políticas de autorização
│   ├── QueryBuilders/        # Query builders Eloquent customizados
│   ├── Resources/            # API Resources do domínio
│   └── States/               # Implementações do State Pattern
├── Shared/
│   ├── Interfaces/           # Contratos compartilhados (ex: DataTransferObjectInterface)
│   └── ...
tests/
├── Unit/                     # Testes unitários (Actions, DTOs, lógica isolada)
├── Feature/                  # Testes de feature (rotas HTTP, fluxo completo)
```

Para criar um novo domain: `php artisan make:domain {NomeDomain}`

---

## DefaultModel (Model Base)

A classe abstrata `DefaultModel` é o model base que todos os models Eloquent do projeto devem estender. Ela fica em `app/Models/DefaultModel.php`.

Serve como ponto único de customização para todos os models. Qualquer comportamento, trait ou configuração adicionada aqui se aplica automaticamente a todos os models. Inclui a trait `HasFactory` por padrão e anotações PHPDoc para métodos estáticos de query, melhorando o autocompletion da IDE.

### O que evitar

- Lógica específica de domínio — mantenha genérico.

### Implementação

```php
namespace App\Models;

use Carbon\Carbon;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

/**
 * @method static Builder query()
 * @method static Builder where(string $column, mixed $operator = null, mixed $value = null, string $boolean = 'and')
 * @method static Builder orWhere(string $column, mixed $operator = null, mixed $value = null)
 * @method static Builder whereIn(string $column, array|mixed $values, string $boolean = 'and', bool $not = false)
 * @method static Builder whereNotIn(string $column, array|mixed $values, string $boolean = 'and')
 * @method static Builder whereNull(string|array $columns, string $boolean = 'and', bool $not = false)
 * @method static Builder whereNotNull(string|array $columns, string $boolean = 'and')
 * @method static Builder whereDate(string $column, string $operator, mixed $value = null, string $boolean = 'and')
 * @method static Builder whereMonth(string $column, string $operator, mixed $value = null, string $boolean = 'and')
 * @method static Builder whereDay(string $column, string $operator, mixed $value = null, string $boolean = 'and')
 * @method static Builder whereYear(string $column, string $operator, mixed $value = null, string $boolean = 'and')
 * @method static static|null find(int|mixed $id, array $columns = ['*'])
 * @method static static findOrFail(int|mixed $id, array $columns = ['*'])
 * @method static static|null first(array $columns = ['*'])
 * @method static static firstOrFail(array $columns = ['*'])
 * @method static static create(array $attributes)
 * @method static static forceCreate(array $attributes)
 * @method static bool insert(array $values)
 * @method static int count(string $columns = '*')
 * @method static paginate(int|null $perPage = null, array $columns = ['*'], string $pageName = 'page', int|null $page = null)
 * @method static simplePaginate(int|null $perPage = null, array $columns = ['*'], string $pageName = 'page', int|null $page = null)
 * @method static static updateOrCreate(array $attributes, array $values = [])
 * @method static delete()
 * @method static destroy(array|int|string $ids)
 * @property Carbon $created_at
 * @property Carbon $updated_at
 */
abstract class DefaultModel extends Model
{
    use HasFactory;
}
```

---

## Models

Os models representam a camada de acesso aos dados e são organizados de acordo com o domínio ao qual pertencem. Ficam em `app/Models/{Domain}/` e devem estender `DefaultModel`.

### Responsabilidades

- Definir atributos via `$fillable` para mass assignment.
- Definir relacionamentos Eloquent (`belongsTo`, `hasMany`, `belongsToMany`, etc.).
- Sobrescrever `newEloquentBuilder()` para utilizar um QueryBuilder customizado quando o domínio tiver um.
- Conter accessors, mutators e casts simples.

### O que evitar

- Implementar regras de negócio nos models — utilize Actions para isso.
- Models extensos. Se um model tem muitos métodos (desconsiderando relationships), reavalie a responsabilidade e extraia para Actions, QueryBuilders ou outras classes do domínio.
- Utilizar `$guarded = []` — sempre defina `$fillable` explicitamente.

### Exemplo

```php
namespace App\Models\Post;

use App\Models\DefaultModel;
use Domain\Post\QueryBuilders\PostQueryBuilder;
use App\Models\User;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Post extends DefaultModel
{
    protected $fillable = [
        'title',
        'author_id',
        'text',
    ];

    // Use 'immutable_datetime' quando quiser CarbonImmutable; 'datetime' retorna Carbon mutável
    protected $casts = [
        'published_at' => 'immutable_datetime',
    ];

    // Override do QueryBuilder
    public function newEloquentBuilder($query): PostQueryBuilder
    {
        return new PostQueryBuilder($query);
    }

    // Relacionamentos
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'author_id');
    }
}
```

---

## Controllers

Os controllers têm como principal objetivo direcionar o fluxo de trabalho: receber a requisição, validar, autorizar e delegar para uma Action. Ficam em `app/Http/Controllers/`.

### Responsabilidades

- Receber e validar requisições HTTP (via injeção de FormRequest).
- Autorizar o usuário via `Gate::authorize()` (padrão Laravel 12+ — o Controller base não inclui mais a trait `AuthorizesRequests`).
- Chamar uma Action para executar a lógica de negócio.
- Retornar a resposta utilizando uma classe Resource.

### O que evitar

- Implementar lógica de negócio diretamente nos controllers — para isso, utilize Actions.
- Interagir com o banco de dados diretamente (nada de `Model::create()` ou `DB::` calls no controller).
- Validar dados inline — sempre utilize FormRequests.
- Utilizar `$this->authorize()` — a trait `AuthorizesRequests` não faz mais parte do Controller base desde o Laravel 11+. Use `Gate::authorize()` ou o middleware `can` via `HasMiddleware`.

### Autorização

Utilize a facade `Gate` para autorização baseada em Policies. Para autenticação, implemente a interface `HasMiddleware` com `auth:sanctum`.

### Retornos para API

O retorno de dados para APIs deve ser feito por meio de Resources. Isso padroniza as respostas e controla quais campos são expostos ao cliente.

### Exemplo

Note que nesse exemplo o controller apenas aplica segurança, filtros e delega para a Action:

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

    // A Action é injetada via type-hint — o container resolve automaticamente
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

## Domain

Os domains são utilizados para organizar lógica específica de um contexto, agrupando classes relacionadas a um único módulo de negócio. Cada domínio é um módulo independente com suas próprias Actions, DTOs, Exceptions, etc.

### Regras gerais

- Cada domínio deve ser independente. Minimize dependências entre domínios.
- Contratos compartilhados ficam em `Domain/Shared/`.

### Estrutura típica

```
Domain/
├── Post/
│   ├── Actions/
│   ├── DataTransferObjects/
│   ├── Collections/
│   ├── Enums/
│   ├── Exceptions/
│   ├── Interfaces/
│   ├── Jobs/
│   ├── Notifications/
│   ├── Policies/
│   ├── QueryBuilders/
│   ├── Resources/
│   └── States/
```

Para criar um novo domain: `php artisan make:domain {NomeDomain}`

> **Nota:** Lembre-se de atualizar a estrutura e documentação sempre que surgir uma nova necessidade ou diretório.

---

### Interfaces compartilhadas

O diretório `Domain/Shared/` contém contratos utilizados por múltiplos domínios. O mais comum é a `DataTransferObjectInterface`, que todo DTO deve implementar:

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

> O contrato cobre os factory methods comuns a todo DTO. Factory methods específicos de um DTO (como `fromModel`) não entram na interface — são declarados apenas no DTO que precisar deles, também retornando `static`.

---

### Comunicação entre domínios

Quando uma Action precisa chamar lógica de outro domínio, ela chama a Action do outro domínio diretamente via injeção no construtor. Como as Actions utilizam `__invoke`, o service container do Laravel resolve as dependências automaticamente — sem necessidade de interfaces ou abstrações extras.

```php
readonly class StorePostAction
{
    // StoreCommentAction (do domínio Comment) injetada via container
    public function __construct(
        private StoreCommentAction $storeCommentAction,
    ) {}

    public function __invoke(PostDTO $postDTO, User $user): Post
    {
        // ... lógica de store do post ...

        if ($postDTO->commentDTO) {
            ($this->storeCommentAction)($postDTO->commentDTO, $post, $user);
        }

        // ...
    }
}
```

Evite criar interfaces que teriam apenas uma única implementação — a chamada direta da Action é preferível para reduzir complexidade desnecessária.

#### Chamada direta vs Evento de domínio

Nem toda comunicação entre domínios deve ser uma chamada direta. Use o critério abaixo para decidir:

- **Injeção direta da Action** quando o domínio de origem **precisa do retorno**, a operação é **síncrona** e faz parte da **mesma transação**. Exemplo: criar um Post e seu Comment juntos — o `StorePostAction` chama o `StoreCommentAction` e depende do resultado dentro da transação.
- **Evento de domínio** quando é um **side effect** que o domínio de origem **não precisa conhecer** — notificar usuários, indexar para busca, gerar analytics. O domínio de origem dispara o evento (ex: `PostPublished`) e quem reage são listeners nos outros domínios.

**Regra de decisão**: se você precisa do valor de retorno e a operação pertence à transação, injete a Action. Se é "aconteceu X, e outros domínios podem querer reagir", dispare um evento. Isso mantém os domínios desacoplados e evita que a dependência entre eles vire um ciclo.

---

## Explicando os diretórios e classes do domain

### Actions (Classes de Serviço)

As Actions encapsulam a lógica de negócio do sistema. Cada Action é responsável por uma única operação específica e bem definida. Ficam em `Domain/{NomeDomain}/Actions/`.

#### Boas práticas

- **Princípio da Responsabilidade Única (SRP)**: Cada classe deve possuir apenas uma finalidade específica, evitando "God Classes". Isso facilita a manutenção, evolução e entendimento do código.
- **`readonly class` com `__invoke()`**: Toda Action deve ser uma `readonly class` com o método `__invoke()` como único método público. Isso permite a injeção automática de dependências pelo service container do Laravel.
- **Transações**: Operações de escrita devem ser envolvidas em `DB::beginTransaction()` / `commit()` / `rollBack()`.
- **DTOs como entrada**: A Action deve receber dados exclusivamente via DTOs, nunca `FormRequest`, `Request` ou arrays crus. O DTO é o contrato de entrada do Domain.
- **Injeção cross-domain**: Actions de outros domínios devem ser injetadas via construtor — o container resolve automaticamente.
- **Reutilização**: Uma mesma Action pode ser chamada de um Controller, Job, Command ou outra Action.

#### Exemplo

```php
namespace Domain\Post\Actions;

use App\Models\Post\Post;
use App\Models\User;
use Domain\Post\DataTransferObjects\PostDTO;
use Domain\Comment\Actions\StoreCommentAction;
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

            // Chamada cross-domain via injeção no construtor
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

Exemplo de utilização no controller:

```php
public function store(StorePostRequest $request, StorePostAction $action): PostResource
{
    Gate::authorize('create', Post::class);

    return PostResource::make(
        $action(PostDTO::fromRequest($request), $request->user()),
    );
}
```

---

### DataTransferObjects (DTOs)

Os DTOs são o **contrato de entrada padrão** da camada de Domain. Eles transportam dados validados e tipados de qualquer fonte da infraestrutura para as Actions. Ficam em `Domain/{NomeDomain}/DataTransferObjects/`.

#### Por que sempre usar DTOs (e não FormRequests diretamente)

Os DTOs desacoplam a camada de Domain da camada de infraestrutura. Uma Action nunca deve receber um `FormRequest`, um `array` cru, ou qualquer objeto específico de HTTP. A camada de infraestrutura (Controller, Job, Command, Event listener) é responsável por construir o DTO e passá-lo para a Action. Isso garante que a mesma Action possa ser reutilizada a partir de qualquer ponto de entrada:

- **HTTP Controller**: `PostDTO::fromRequest($request)`
- **Artisan Command**: `PostDTO::fromArray($input)`
- **Job / Queue**: `PostDTO::fromArray($payload)`
- **Event Listener**: `PostDTO::fromModel($event->post)`
- **Outra Action**: passa o DTO diretamente

#### Regras

- Deve implementar `DataTransferObjectInterface`.
- Deve ser declarado como `final readonly class`.
- Não deve conter lógica de negócio.
- Factory methods (`fromRequest`, `fromArray`, `fromModel`) devem retornar `static`.

#### Exemplo

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

Com os arrays validados no FormRequest do controller, a chamada do DTO fica simples:

```php
PostDTO::fromRequest($request)
```

---

### Collections

As Collections no Laravel fornecem uma maneira poderosa e fluida de manipular conjuntos de dados, estendendo funcionalidades além das arrays nativas do PHP. Ficam em `Domain/{NomeDomain}/Collections/`.

> Para mais detalhes, consulte a [documentação oficial do Laravel](https://laravel.com/docs/eloquent-collections).

---

### Enums

Os Enums definem valores fixos e imutáveis dentro de um domínio. Ficam em `Domain/{NomeDomain}/Enums/`.

Regras:

- Deve usar PHP backed enums (`string` ou `int`).
- Utilize para campos de status, tipos, categorias — qualquer conjunto fixo de valores.

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

### Exceptions

As Exceptions específicas de cada domínio devem ser criadas em `Domain/{NomeDomain}/Exceptions/`.

Dicas:

- Nomeie de forma clara e descritiva, como `PostNotFoundException` ou `InvalidPostStateException`.
- Sempre inclua mensagens explicativas e, se necessário, códigos de erro ou contexto adicional.
- A escolha do tipo de exception é por contexto de execução: para erros no runtime da API, estenda `DomainHttpException` (que herda da `HttpException` do Symfony e é renderizada automaticamente pelo Laravel); para erros fora do ciclo HTTP, como jobs de segundo plano, estenda `DomainException` (plain). Ambas as bases ficam em `Domain/Shared/Exceptions/`.
- Detalhes e exemplos na seção [Tratamento de Erros e Respostas da API](#tratamento-de-erros-e-respostas-da-api).

---

### QueryBuilders

Os QueryBuilders são utilizados para manipular as consultas ao banco de dados de forma personalizada e eficiente, encapsulando regras e filtros específicos. Ficam em `Domain/{NomeDomain}/QueryBuilders/`.

Para substituir o builder padrão do Eloquent em um model, sobrescreva o método `newEloquentBuilder()`. O uso de DTOs como parâmetro garante consistência e validação dos dados utilizados na consulta.

#### Exemplo

No model:

```php
class Post extends DefaultModel
{
    public function newEloquentBuilder($query): PostQueryBuilder
    {
        return new PostQueryBuilder($query);
    }
}
```

Na classe QueryBuilder:

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

Com o método sobrescrito no model, é possível chamar os métodos personalizados de forma estática:

```php
public function index(SearchPostRequest $request): AnonymousResourceCollection
{
    Gate::authorize('viewAny', Post::class);

    $builder = Post::search(PostSearchDTO::fromRequest($request));
    $builder->orderBy('created_at');
    $builder->with(['author']);

    return PostResource::collection($builder->cursorPaginate());
}
```

#### Por que QueryBuilder e não Repository

Esta arquitetura **não utiliza o padrão Repository** — e isso é uma decisão deliberada. Em um ORM como o Eloquent, uma camada de Repository é redundante e prejudicial. O QueryBuilder customizado é a forma correta de encapsular o acesso a dados aqui.

**O que o Repository tentaria resolver — e por que não compensa:**

- **"Abstrair o banco de dados"**: o Eloquent **já é** essa abstração. Ele suporta múltiplos drivers (PostgreSQL, MySQL, SQLite, SQL Server) por configuração. Colocar um Repository por cima é abstrair o que já é uma abstração — uma camada sobre a camada.
- **"Permitir trocar a fonte de dados"**: na prática isso quase nunca acontece, e quando acontece o Eloquent já cobre. A interface de Repository quase sempre tem **uma única implementação** (a Eloquent), violando o princípio de só criar abstração quando há real necessidade de troca — o mesmo critério que aplicamos para evitar interfaces de implementação única nas Actions.
- **"Centralizar as queries"**: é exatamente o papel do QueryBuilder. A diferença é que o QueryBuilder **estende** o Eloquent em vez de **escondê-lo**.

**O custo de adotar Repository sobre um ORM:**

- **Camada anêmica de passthrough**: a maioria dos métodos vira `return Post::find($id)`, sem agregar nada além de indireção.
- **Perda do poder do Eloquent**: relationships, lazy/eager loading, scopes, paginação, `chunk`, `cursor`. Ou você expõe o builder (e o Repository perde o sentido), ou reimplementa tudo na interface (trabalho enorme e pior).
- **Explosão de métodos**: `findById`, `findByTitle`, `findByAuthorAndStatus`... a interface cresce sem controle. O QueryBuilder resolve isso com filtros compostos a partir de um DTO (veja o `search()` acima).

**Resumindo**: o QueryBuilder customizado entrega tudo que se espera de um Repository — queries reutilizáveis, encapsuladas e testáveis — sem a indireção redundante, abraçando o Eloquent em vez de lutar contra ele. Acesso a dados que precise de filtros complexos vai para o QueryBuilder; orquestração e regra de negócio ficam na Action.

---

### Jobs

Diretório reservado para classes de processamento em segundo plano (background jobs). Ficam em `Domain/{NomeDomain}/Jobs/`.

Boas práticas:

- Cada job deve ter uma única responsabilidade clara e específica.
- Garanta que os jobs sejam idempotentes sempre que possível (seguros para retry).
- Inclua logs para rastrear falhas e sucessos.
- Lógica pesada deve ser delegada para Actions.

> Para mais detalhes, consulte a [documentação oficial do Laravel](https://laravel.com/docs/queues).

---

### Notifications

Diretório para classes de notificações enviadas aos usuários (email, SMS, push). Ficam em `Domain/{NomeDomain}/Notifications/`.

> Para mais detalhes, consulte a [documentação oficial do Laravel](https://laravel.com/docs/notifications).

---

### Policies

Diretório para políticas de autorização — regras que determinam se um usuário pode executar determinada ação no sistema. Ficam em `Domain/{NomeDomain}/Policies/`.

> Para mais detalhes, consulte a [documentação oficial do Laravel](https://laravel.com/docs/authorization).

---

### Resources

Os Resources são responsáveis por padronizar as respostas das APIs. Devem ser utilizados sempre que o controller precisar enviar dados para o cliente. Ficam em `Domain/{NomeDomain}/Resources/`.

Regras:

- Deve ser utilizado para todas as respostas de API.
- Deve definir explicitamente quais campos são retornados — nunca retorne models crus.
- Controle a exposição de campos sensíveis aqui.

> Para mais detalhes, consulte a [documentação oficial do Laravel](https://laravel.com/docs/eloquent-resources).

---

### States (State Pattern)

Os States implementam o State Pattern para gerenciar comportamentos de uma entidade com base em seu estado atual. Ficam em `Domain/{NomeDomain}/States/`.

#### Quando usar States vs Enums

Use o **State Pattern** quando a troca de status carrega uma responsabilidade significativa — side effects, validações, notificações, mudanças de permissão, ou orquestração de múltiplas operações. Cada estado vira uma classe que encapsula a regra da transição (para qual status ir a partir dali), enquanto a Action que recebe o estado aplica a persistência e os side effects.

Use um **Enum simples** (com validação na Action) quando o status é apenas um label sem comportamento atrelado — como um campo `priority` (`low`, `medium`, `high`) ou um campo `category`.

**Regra de decisão**: Se mudar um status exige mais do que atualizar uma coluna (ex: dispara emails, muda permissões, cria audit logs, bloqueia/permite outras operações), use States. Se é apenas trocar um valor, um Enum validado na Action resolve.

Vale notar que os dois **se compõem**: o status persistido é um backed enum (ex: `PostStatus`) e cada classe de State retorna o valor do enum correspondente à sua transição. O enum é o dado (o valor gravado na coluna); o State é o comportamento que decide o próximo status.

#### Regras

- Cada estado é uma classe separada implementando uma interface compartilhada de estado (ex: `PostStatusState`).
- O único método público (`handle()`) **retorna o enum de status** correspondente àquela transição.
- O estado é **puro**: não recebe o model, não toca no banco e não dispara side effects. Ele apenas responde "para qual status essa transição leva". Regras de guarda (transição inválida → exception) podem viver dentro do `handle()` quando necessário.
- Estados são `readonly class` e nomeados conforme o status que representam (`PublishedState`, `ArchivedState`, etc.).
- Quando a entidade tem mais de um eixo de status, agrupe os estados por contexto em subpastas, cada grupo com sua própria interface (ex: `States/Bank/` e `States/Card/` para uma mesma entidade). Com um único eixo, ficam direto em `States/`.
- A **Action recebe a interface do estado**, chama `handle()` para obter o novo status e aplica a persistência (e eventuais side effects) dentro da transação.

#### Exemplo

Reaproveitando o backed enum `PostStatus` (`Draft`, `Published`, `Archived`) definido na seção de Enums.

A interface compartilhada do estado, cujo `handle()` retorna o enum:

```php
namespace Domain\Post\Interfaces;

use Domain\Post\Enums\PostStatus;

interface PostStatusState
{
    public function handle(): PostStatus;
}
```

Cada estado retorna o status que representa, em `States/`:

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

A Action recebe a interface do estado, obtém o novo status e persiste dentro da transação:

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

O ponto de entrada escolhe o estado concreto e o entrega à Action:

```php
($action)($post, new PublishedState(), $request->user());
```

---

## FormRequests

Classes de validação para dados de requisição HTTP. Ficam em `app/Http/Requests/`.

Regras:

- Deve ser utilizado em todos os métodos do controller que aceitam input do usuário.
- Deve definir `rules()` retornando as regras de validação.
- Pode definir `authorize()` retornando um boolean (ou delegar para Policies).
- Controllers nunca devem validar inline — sempre injete um FormRequest.

### Exemplo

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Autorização tratada via Policy no controller
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

## Tratamento de Erros e Respostas da API

A escolha do tipo de exception é por **contexto de execução**:

- **HTTP Exceptions** — para erros que ocorrem no **runtime de uma requisição da API**. Crie exceptions de domínio que estendem a base `DomainHttpException` (que por sua vez estende a `HttpException` do Symfony) — assim você tem nomes semânticos (`PostNotFoundException`) com o render automático do Laravel, no status correto. Para casos genéricos também é válido lançar as exceptions HTTP do Symfony diretamente (`NotFoundHttpException`, etc.) ou usar o helper `abort()`. Em todos os casos **não é preciso modificar o handler**.
- **Domain Exceptions** — para erros de regra de negócio que ocorrem **fora do ciclo HTTP**, tipicamente em **jobs de segundo plano**. Como não há resposta HTTP a montar, elas carregam apenas semântica de domínio (mensagem e contexto) e são tratadas pela infraestrutura de filas (logs, `failed_jobs`, retries/backoff).

### Formato padrão de resposta de erro (API)

A API segue o formato padrão do Laravel para respostas JSON de erro — não inventamos um envelope próprio:

```json
// Erro de validação (422)
{
    "message": "The title field is required.",
    "errors": {
        "title": ["The title field is required."]
    }
}

// Demais erros (404, 403, 401, 5xx)
{
    "message": "Post não encontrado."
}
```

O Laravel já entrega esse formato automaticamente quando a requisição espera JSON (`Accept: application/json` ou rotas de API): `ValidationException` → 422, `AuthenticationException` → 401, `AccessDeniedHttpException` → 403, `ModelNotFoundException` / `NotFoundHttpException` → 404. Em produção (`APP_DEBUG=false`), detalhes internos e stack traces nunca são expostos. Por isso, **o handler em `bootstrap/app.php` não precisa de customização** para o fluxo de API.

### HTTP Exceptions (runtime da API)

Crie uma base `DomainHttpException` em `Domain/Shared/Exceptions/` estendendo a `HttpException` do Symfony. Por herdar de `HttpException`, o Laravel já a renderiza no formato JSON correto com o status que ela carrega — sem nenhuma configuração no handler. A base existe para dar um tipo comum (catchável) às exceptions HTTP de domínio e centralizar comportamento compartilhado:

```php
namespace Domain\Shared\Exceptions;

use Symfony\Component\HttpKernel\Exception\HttpException;

abstract class DomainHttpException extends HttpException
{
}
```

Cada exception concreta fixa o status HTTP e uma mensagem padrão:

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

Lance onde o erro acontece — o Laravel cuida da resposta:

```php
$post = Post::find($id) ?? throw new PostNotFoundException();
```

Isso resulta em `404` com `{ "message": "Post não encontrado." }`, sem nenhuma configuração adicional.

### Domain Exceptions (jobs de segundo plano)

Crie uma base semântica em `Domain/Shared/Exceptions/` (sem status HTTP, já que não há resposta a renderizar) e estenda-a por domínio:

```php
namespace Domain\Shared\Exceptions;

use RuntimeException;

abstract class DomainException extends RuntimeException
{
}
```

```php
namespace Domain\Post\Exceptions;

use Domain\Shared\Exceptions\DomainException;

class PostProcessingFailedException extends DomainException
{
}
```

Em um job, lançar a exception marca o job como falho — o Laravel registra em `failed_jobs` e aplica a política de retry/backoff configurada. Não há render nem resposta HTTP envolvida:

```php
namespace Domain\Post\Jobs;

use App\Models\Post\Post;
use Domain\Post\Exceptions\PostProcessingFailedException;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class ProcessPostJob implements ShouldQueue
{
    use Queueable;

    public function __construct(
        private readonly int $postId,
    ) {}

    public function handle(): void
    {
        $post = Post::find($this->postId)
            ?? throw new PostProcessingFailedException(
                __('post.processing.not_found', ['id' => $this->postId]),
            );

        // ... processamento ...
    }
}
```

---

## Testes

Tecnologia: [Pest PHP](https://pestphp.com/)

### Estrutura

- **Testes unitários** (`tests/Unit/`): Testam Actions, DTOs e lógica isolada diretamente.
- **Testes de feature** (`tests/Feature/`): Testam rotas HTTP de ponta a ponta, verificando respostas e estado do banco.

### Padrão: Arrange → Act → Assert

Todo teste deve seguir essa estrutura:

1. **Arrange (Preparação)**: Configura o estado inicial — factories, DTOs, mocks.
2. **Act (Execução)**: Executa o comportamento sendo testado.
3. **Assert (Verificação)**: Verifica se o resultado é o esperado.

### Regras

- Factories devem espelhar a utilização real o mais próximo possível.
- Use `->group()` para categorizar testes por tipo e domínio (ex: `'Unit', 'Post'`).
- Use datasets (`->with()`) para testes parametrizados.

### Exemplo teste unitário

Diretório: _/tests/Unit/_

Nesse teste iremos testar somente a Action e seu retorno:

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

### Exemplo teste feature

Diretório: _/tests/Feature/_

Nesse teste iremos testar as rotas da API e se o banco foi devidamente populado/modificado:

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
