DDD guideline

Terminology
Domain: a sphere of knowledge, influence or activity around which the application logic resolves.

Subdomains: a subset of the domain to divide the entire complexity of the company’s domain into smaller parts. e.g. Catalog, Customer data platform (CDP), Purchasing, Inventory and so on.

Bounded context: is simply boundary within a domain where a particular domain models applies.

Context map: it’s a map to relate different bounded context together to identify the relationship and translation of data across from one model to the other subsystem. Types of context maps:
* Partnership: The team have a mutual dependency on each other for delivery.
* Shared kernel: both referenced and owned by multiple bounded contexts, each team is free to modify the compiled library that defines the integration contract.
* Customer-supplier: Upstream and downstream relationship.
* Conformist: The downstream team can accept the upstream team’s model.
* Open host service (OHS) / Published language (PL): The upstream supplier decouples its implementation model from the public interface to protect the consumers from changes. 
* Anticorruption-layer (ACL): Downstream bounded context is not willing to conform the upstream, instead translate the upstream bounded context’s model into a model.
* Separate ways: not collaborate at all.
Checkout out for more details

Repository: A mechanism for encapsulating storage, retrieval, and search behavior which emulates a collection of objects.

Factory: A mechanism for encapsulating complex creation logic and abstracting the type of a created object for the sake of a client.

Ubiquitous language: A language of a model such that each term is unambiguous and no rules contradict.

Architecture
The typical enterprise application architecture consists of the following four conceptual layers:
* User interface (presentation layer): responsible for presenting information to the user and interpreting user commands, e.g. UI views, controllers.
    * This could be API that is responsible for the navigation between UI screen in the application as well as the interaction with the application layers of other system.
    * Has models or DTO definitions of its own with attributes relative to its layer. If this is an API then model/DTO have attributes for formatting or data type validations.
    * This doesn’t contain any business or domain related logic or data access logic.
* Application layer: The layer coordinates the application activity. It doesn’t contain any business logic, it doesn’t hold the state of business objects, but it can hold the state of an application task’s progress, e.g. Application service, components.
    * Perform the basic (non-business related) validation on the user input data before transmitting it to the other layers of the application.
    * Contain transaction management (unit of work).
    * Bridge between presentation layer and domain layer.
* Domain layer: Contains the information about the business domain, persistence of the business objects and possibly their state is delegated to the infrastructure layer, e.g. Domain service, aggregate entities, value objects.
    * Domain model includes aggregates, entities and value objects.
    * Note don’t put DTO in the domain layer since domain layer doesn’t care about things outside of its own.
    * Responsible for the concept of business domain, information about the business use case and the business rules, and domain objects encapsulate the state and behavior of business entities.
* Infrastructure layer: The layer acts as a supporting library for all the other layers, it implements for business objects, contains supporting libraries for the user interface layer, e.g. Repository.
    * Database persistence store
    * Libraries you define this layer as infrastructure code
    * Mail or notification service
￼

Example
Imagine a product class in the logistics domain, for tracking around the warehouse you need a barcode, for shipping you need to packaged dimensions and weight. Think a product with photos, description and specs for display on an e-commerce website.

APIs
* GET /api/v1/products
* POST /api/v1/products
* PUT /api/v1/products/{product_id}

Data objects
Example `POST /api/v1/products`
```json
{
  "name": "foo",
  "description": "test",
  "image_url": "https://images.mamilove.com.tw/brand/0eb36102e3-1574845513.jpeg",
  "barcode": "abc-abc-1234",
  "specs": [
    {
	"id": 1,
	"name": "style",
	"value": "balance"
    }
  ]
}
```

```php
// Under routes/admin.php
Route::group([‘prefix’ => ‘/products’], function () {
	Route::get(‘/‘, ‘AdminApp\Modules\Product\ProductController@get’);
	Route::post(‘/‘, ‘AdminApp\Modules\Product\ProductController@create’);
	Route::put(‘/{product_id}‘, ‘AdminApp\Modules\Product\ProductController@update’);
});
```
```php
// Presentation layer
// This is an API entrypoint that acts as a bridge between user interaction and application layer presents information to the client.
use AdminApp\Modules\Product\Services\FetchProductService;

class ProductController 
{
	public function get(Request $request, FetchProductService $service, ProductTransformer $transformer)
	{
		$request->validate([
			‘ids’ => ‘array’,
			‘name’ => ‘string’,
			‘barcode’ => ‘string’,
			‘limit’ => ‘int|default:5’,
			‘offset’ => ‘int|default:5’,
		]);

		return $transformer->transform($service->execute($request->ids, $request->name, $request->barcode, $request->limit, $request->offset));
	}

	public function create(ProductRequest $request, CreateProductService $service)
	{
		$request->validate();

		$service->execute($request->dto());
	}

	public function update(int $product_id, ProductRequest $request, UpdateProductService $service)
	{
		$request->validate();

		$service->execute($request->dto());
	}
}
```

```php
// Application layer
use Modules\Logistics\Infrastructure\ProductRepositoryInterface;

class FetchProductService
{
	public function __construct(ProductRepositoryInterface $product_repo)
	{
		$this->product_repo = $product_repo;
	}

	public function execute(array $ids, string $name, string $barcode, int $limit, int $offset)
	{
		return $this->product_repo->get($dto->ids, $dto->name, $dto->barcode, $limit, $offset);
	}
}

use Illuminate\Database\Connection;
use Modules\Logistics\Entities\Product;
use Modules\Logistics\Infrastructure\ProductRepositoryInterface;

class CreateProductService
{
	public function __construct(Connection $connection, ProductRepositoryInterface $product_repo)
	{
		$this->connection = $connection;
		$this->product_repo = $product_repo;
	}

	public function execute(ProductDto $dto)
	{
		// Build an aggregate root.
		$product = new Product($dto->name, $dto->description, $dto->image_url, $dto->barcode);
		// Validate the attributes against the business rules.
		$product->validated();
		$product->dispenceSKUs();

		$this->connection->beginTransaction();

		try {
			$this->product_repo->save($product);
			$this->connection->commit();
		} catch (Exception $e) {
			$this->connection->rollBack();
			throw $e;
		}
	}
}

use Illuminate\Database\Connection;
use Modules\Logistics\Entities\Product;
use Modules\Logistics\Infrastructure\ProductRepositoryInterface;

class UpdateProductService
{
	public function __construct(Connection $connection, ProductRepositoryInterface $product_repo)
	{
		$this->connection = $connection;
		$this->product_repo = $product_repo;
	}

	public function execute(int $product_id, ProductDto $dto)
	{
		$product = $this->product_repo->getById($product_id);

		// Throw not found exception when the given product does not exist followed by fail fast principle.
		if (! $product_id) {
			throw new ProductNotFoundException();
		}

		$product->setName($dto->name)
			->setDescription($dto->description)
			->setImageUrl($dto->image_url)
			->setBarcode($dto->barcode);

		// Validate the attributes against the business rules.
		$product->validated();
		$product->dispenceSKUs();

		$this->connection->beginTransaction();

		try {
			$this->product_repo->save($product);
			$this->connection->commit();
		} catch (Exception $e) {
			$this->connection->rollBack();
			throw $e;
		}
	}
}
```

```php
// Domain layer

// Aggregate root
class Product
{
	private $product_id;
	
	private $name;

	private $description;

	private $image_url;

	private $barcode;

	private $size;

	public function __construct(
		string $name,
		string $description,
		string $image_url,
		string $barcode
	) {
		$this->name = $name;
		$this->description = $description;
		$this->image_url = $image_url;
		$this->barcode = $barcode;
	}

	public function setSize(Size $size)
	{
		return $this->size = $size;
	}

	public function validated()
	{
		$this->validator->validate([]);
	}

	public function dispenseSKUs()
	{
		
	}
}

// Entity
class SaleItem
{
}
```

```php
// Infrastructure layer

class ProductRepositoryInterface
{
	public function getById(int $product_id);

	public function save(Product $product);
}

class ProductRepository implements ProductRepositoryInterface
{
	public function getById(int $product_id)
	{
		return $this->em->find($product);
	}

	public function save(Product $product)
	{
		$this->em->persist($product);
	}
}
```

Avoid anemic model for domain objects


Adapter
The incoming request body converts into DTO object

Domain service
Domain service carry domain knowledge, application service don’t, it holds domain logic that doesn’t naturally fit entities and value objects.

https://enterprisecraftsmanship.com/posts/domain-vs-application-services/

CQRS
Command and query responsibility segregation (CQRS) separates the models for reading and writing data.
* Queries: return a result and do not change the state of the system, and they’re free of side effect.
* Command: change the state of a system but do not return a value
The valuable idea in the principle is that it’s extremely handy if you can clearly separate methods that change state from those that don’t -  Command Query Separation (CQS).
￼
Event sourcing
Event sourcing is a pattern defines an approach to handling operations on data that’s driven by a sequence of current append-only store. Application code sends a series of events that imperatively describe each action that has occurred on the data to the event store where they ‘re persisted. Gives a new way of persisting application state as an ordered sequence of events.


Additional references
https://devonburriss.me/ddd-glossary/

https://youtu.be/aQVSzMV8DWc
