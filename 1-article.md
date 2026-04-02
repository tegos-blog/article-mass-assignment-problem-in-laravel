Mass assignment in Laravel is one of those things that feels like magic at first.
You see it in every tutorial: just toss your validated data into `Model::create($data)` or `$model->update($request->validated())`, set up your `$fillable`, and you're off to the races. For quick projects? It works. But when your app starts to get bigger, what once felt convenient can start to cause real trouble.

## The Usual Approach

Let's be real-most controllers start out pretty much like this:

```php
// OrderController.php
public function store(OrderStoreRequest $request): OrderResource
{
    $order = Order::query()->create($request->validated());

    return OrderResource::make($order);
}
```

It's clean, it's simple, and it just works. You throw whatever new fields you need into `$fillable`, and data slides right in.

But your `Order` model slowly grows. You add billing fields, delivery options, status flags, internal data. Suddenly `$fillable` looks like this:

```php
protected $fillable = [
    'user_id', 'manager_id', 'number',
    'price_type_id', 'contract_code', 'currency_id',
    'address_id', 'status_id', 'direction_id', 'direction_type_id',
    'deliver_together', 'is_forced_processing', 'price', 'comment',
    'courier_date', 'courier_time', 'phone', 'first_name', 'last_name',
    'ip', 'user_agent', 'geo_location', 'finance_condition_data', 'is_from_api',
];
```

Twenty-four fields. Now ask yourself: what does `Order::create($request->validated())` actually write to the database when called from the customer API? What about from the admin panel? From an internal backend action?

You don't know without digging into the request class, the validation rules, and then mentally subtracting fields that don't come from that particular endpoint. That's the real problem.

## What Goes Wrong in Practice

### You lose visibility fast

When you have three different endpoints that create an order - customer-facing, frontend panel, admin - each sets different fields. With mass assignment, the controller tells you nothing. You have to trace the full chain: controller → FormRequest → `$fillable` → model. And even then, you're guessing which fields actually come in from each context.

A new developer joins, gets a ticket "order is created with wrong status", searches for where `status_id` is set on `Order`... and finds nothing obvious. Because it comes in somewhere inside `$request->validated()`.

### The real problem: `$fillable` has no context

Here's the deeper issue. It's not really about how many fields are in `$fillable`. It's about what those fields mean and who is allowed to set them.

Look at the `Order` model again:

```php
protected $fillable = [
    'is_forced_processing', // only admins
    'is_from_api',          // only customer API
    'manager_id',           // only internal, backend logic
    'status_id',            // should only be set internally
    'price',                // computed automatically
    // ...20 more
];
```

`$fillable` is a flat list. It has no idea who is calling `Order::create()`. It doesn't know if it's the admin panel, the customer API, or an internal action. Every field in that list is equally "allowed" for everyone.

So what do you put in `$fillable`? If you add `is_forced_processing` - it becomes fair game for any endpoint that calls `Order::create($data)`. If you leave it out - your admin action has to use `fill()` + `save()` or `Query::update()` to bypass the guard. Either way, you're working around the model instead of working with it.

`$fillable` protects against fields that are not in the list. It doesn't protect against fields that are in the list but shouldn't be settable from a specific context. That's a different kind of protection - and it belongs at the endpoint level, not the model level.

## What I Use Instead

The pattern that actually works: **DTO + Action + explicit field mapping**.

The controller converts the request to a typed DTO:

```php
// OrderController.php (Frontend)
public function store(
    OrderStoreRequest $request,
    OrderCreateAction $orderCreateAction,
): OrderResource {
    $orderCreateDTO = OrderCreateDTO::fromRequest($request);
    $order = $orderCreateAction->handle($orderCreateDTO, $user->id);

    return OrderResource::make($order);
}
```

The DTO maps exactly what this endpoint cares about:

```php
final class OrderCreateDTO
{
    public string $firstName;
    public string $lastName;
    public string $phone;
    public bool $deliverTogether;
    public ?int $directionId = null;
    public ?string $comment = null;
    public ?string $ip = null;

    public static function fromRequest(OrderStoreRequest $request): self
    {
        $self = new self();
        $self->firstName = $request->safe()->input('first_name');
        $self->lastName = $request->safe()->input('last_name');
        $self->phone = $request->safe()->input('phone');
        // ... etc, explicitly mapping
        $self->ip = $request->ip(); // this isn't user input :)
        return $self;
    }
}
```

The Action uses the DTO and writes exactly what it needs to:

```php
final class OrderCreateAction implements Actionable
{
    public function handle(OrderCreateDTO $dto, int $userId): Order
    {
        // ... business logic: cart check, conditions, geo IP ...

        $order = Order::query()->create([
            'user_id' => $userId,        // from auth
            'status_id' => $statusEnum->value, // computed
            'price_type_id' => $userPriceType->id, // from profile
            'first_name' => $dto->firstName,
            'last_name' => $dto->lastName,
            'phone' => $dto->phone,
            'ip' => $dto->ip,
            // ...
        ]);

        return $order;
    }
}
```

Now it's obvious: `user_id` comes from auth, `status_id` is computed internally, `first_name` comes from the DTO. You don't have to trace three files to understand what the endpoint writes.

And the admin `OrderCreateAction` is a separate class with its own DTO - it sets different fields, without any chance of leaking between contexts.

## The "Just Disable It" Way

Some devs go a different route: skip `$fillable` entirely and disable mass assignment protection globally. Nuno Maduro included this as an opt-in feature in his [essentials](https://github.com/nunomaduro/essentials) package - the idea is that `$fillable` gives a false sense of security and the real protection should happen at the validation layer, not the model layer.

Usually, you add this to your `AppServiceProvider`:

```php
Model::unguard();
```

Or using `$guarded = []` on the model itself. Now every field is fillable, no maintenance of a list required.

This is a valid opinion. If you trust your FormRequests to filter input correctly, the model-level guard is indeed redundant. But it requires discipline: every endpoint must validate strictly, every new field must be accounted for in every request that touches the model. One missed `$request->all()` and anything goes through.

For me, unguarding is trading one problem (maintaining `$fillable`) for a bigger one (trusting every developer on the team to never pass unvalidated data to `create()`). In a large team, that's a lot of trust.

## The Point

Mass assignment isn't broken. For simple CRUD, it's totally reasonable. But when your model has 20+ fields, multiple creation contexts, and a team of developers - explicit is better than magic.

The question to ask: can you open a controller method and immediately know what fields get written to the database? If the answer is "no, I need to check `$fillable` and the FormRequest and hope nothing slips through" - that's the problem.

DTOs and Actions add some boilerplate. But they make the code honest: each endpoint writes exactly what it says it writes, nothing more.

## TL;DR

- Mass assignment with `$fillable` is fine for small, simple apps.
- When things grow and your models pile up the fields, you lose track of what's really getting written.
- The issue isn't just field count; it's that `$fillable` has zero context. Anyone can set anything on accident.
- Use a DTO per context to map request fields explicitly, and an Action to write to the model with explicit field names.
- The best test: when you look at a controller, do you know-without checking three files-exactly what's hitting the database? With mass assignment, probably not. With explicit mapping, you always do.

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Notes from real-world Laravel.**