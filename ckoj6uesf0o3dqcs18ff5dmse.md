## Laravel Service Classes Explained

Benjamin Franklin once said —

> A place for everything, everything in its place.

This applies to software development as well. Understanding which portion of the code goes where is the key to a maintainable code base.

[Laravel](https://laravel.com/) being an elegant web framework, comes with a pretty organized [directory structure](https://laravel.com/docs/8.x/structure) by default but still I've seen a lot of people suffer.

Don't get me wrong. It's a no brainer that controllers go inside the `controllers` directory, no confusions whatsoever. The thing people often confuse themselves with is, what to write in a controller and what not to.

# Table of Content

- [Project Codes](#project-codes)
- [The Scenario](#the-scenario)
- [Understanding Business Logic](#understanding-business-logic)
- [Service Classes to the Rescue](#service-classes-to-the-rescue)
- [Action Re-usability](#action-re-usability)
- [Closing Thoughts](#closing-thoughts)

# Project Codes

You can find an implementation of the service discussed in this article in the following repository:

%[https://github.com/fhsinchy/laravel-livewire-shopping-cart]

Apart from Laravel, the project makes use of [Livewire](https://laravel-livewire.com/) and [TailwindCSS](https://tailwindcss.com/).

# The Scenario

Take the following piece of code for example:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class CartItemController extends Controller
{
    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
        $content = session()->has('cart') ? session()->get('cart') : collect([]);
        $total = $content->reduce(function ($total, $item) {
            return $total += $item->get('price') * $item->get('quantity');
        });

        return view('cart.index', [
            'content' => $content,
            'total' => $total,
        ]);
    }

    /**
     * Store a newly created resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request)
    {
        $request->validate([
            'name' => 'required|string',
            'price' => 'required|numeric',
            'quantity' => 'required|integer',
        ]);

        $cartItem = collect([
            'name' => $request->name,
            'price' => floatval($request->price),
            'quantity' => intval($request->quantity),
            'options' => $request->options,
        ]);

        $content = session()->has('cart') ? session()->get('cart') : collect([]);

        $id = request('id');

        if ($content->has($id)) {
            $cartItem->put('quantity', $content->get($id)->get('quantity') + $request->quantity);
        }

        $content->put($id, $cartItem);

        session()->put('content', $content);

        return back()->with('success', 'Item added to cart');
    }

    /**
     * Display the specified resource.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function show($id)
    {
        $content = session()->get('cart');

        if ($content->has($id)) {
            $item = $content->get($id);

            return view('cart', compact('item'));
        }

        return back()->with('fail', 'Item not found');
    }

    /**
     * Update the specified resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, $id)
    {
        $content = session()->get('cart');

        if ($content->has($id)) {
            $cartItem = $content->get($id);

            switch ($request->action) {
                case 'plus':
                    $cartItem->put('quantity', $content->get($id)->get('quantity') + 1);
                    break;
                case 'minus':
                    $updatedQuantity = $content->get($id)->get('quantity') - 1;

                    if ($updatedQuantity < 1) {
                        $updatedQuantity = 1;
                    }

                    $cartItem->put('quantity', $updatedQuantity);
                    break;
            }

            $content->put($id, $cartItem);

            session()->put('cart', $content);

            return back()->with('success', 'Item updated in cart');
        }

        return back()->with('fail', 'Item not found');
    }

    /**
     * Remove the specified resource from storage.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function destroy($id)
    {
        $content = session()->get('cart');

        if ($content->has($id)) {
            session()->put('cart', $content->except($id));

            return back()->with('success', 'Item removed from cart');
        }

        return back()->with('fail', 'Item not found');
    }
}
```

This is a controller from an imaginary e-commerce project, responsible for managing the shopping cart. Although this is a perfectly valid piece of code, there are some problems.

The controller also knows too much. Look at the `index` method for example. It doesn't need to know whether the cart content comes from the session or database. Neither does it need to know how to calculate the total price of the cart items.

A controller should be only responsible for transporting requests and responses. The inner details aka the business logic should be handled by other classes in the server.

# Understanding Business Logic

As explained in this [thread](https://softwareengineering.stackexchange.com/questions/234251/what-really-is-the-business-logic):

> Business logic or domain logic is that part of the program which encodes the real-world business rules that determine how data can be created, stored, and changed. It prescribes how business objects interact with one another, and enforces the routes and the methods by which business objects are accessed and updated.

In case of a simple shopping cart system, the business logic behind adding an item to cart can be described as follows:

1. Take necessary product information (id, name, price, quantity) as input.
2. Validate the input data.
3. Form a new cart item.
4. Check if the item already exists in the cart.
5. If yes, update it's quantity and if no, add the newly formed item to cart.

Now lets have a look at the `store` method:

```php
    /**
     * Store a newly created resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(Request $request)
    {
        // validate the input data.
        $request->validate([
            'name' => 'required|string',
            'price' => 'required|numeric',
            'quantity' => 'required|integer',
        ]);

        // form a new cart item.
        $cartItem = collect([
            'name' => $request->name,
            'price' => floatval($request->price),
            'quantity' => intval($request->quantity),
            'options' => $request->options,
        ]);

        // check if the item already exists in the cart.
        $content = session()->has('cart') ? session()->get('cart') : collect([]);
        $id = request('id');
        if ($content->has($id)) {
            // if yes, update it's quantity
            $cartItem->put('quantity', $content->get($id)->get('quantity') + $request->quantity);
        }

        // if no, add the newly formed item to cart.
        $content->put($id, $cartItem);
        $this->session->put('content', $content);

        return back()->with('success', 'Item added to cart');
    }
```

As you can see, the business logic translates to code pretty accurately. Now the problem is, controllers are not meant to contain business logic.

# Service Classes to the Rescue

According to the very popular [alexeymezenin/laravel-best-practices](https://github.com/alexeymezenin/laravel-best-practices/) repository:

![Business logic should be in service class](https://cdn.hashnode.com/res/hashnode/image/upload/v1620657955936/1AfO4cLA6.png)

The idea of service classes is not something built into the framework or documented in the official docs. As a result, different people refer to them differently. At the end of the day, service classes are plain classes responsible for holding the business logic.

A service class for holding the shopping cart related business logic can be as follows:

```php
<?php

namespace App\Services;

use Illuminate\Support\Collection;
use Illuminate\Session\SessionManager;

class CartService {
    const MINIMUM_QUANTITY = 1;
    const DEFAULT_INSTANCE = 'shopping-cart';

    protected $session;
    protected $instance;

    /**
     * Constructs a new cart object.
     *
     * @param Illuminate\Session\SessionManager $session
     */
    public function __construct(SessionManager $session)
    {
        $this->session = $session;
    }

    /**
     * Adds a new item to the cart.
     *
     * @param string $id
     * @param string $name
     * @param string $price
     * @param string $quantity
     * @param array $options
     * @return void
     */
    public function add($id, $name, $price, $quantity, $options = []): void
    {
        $cartItem = $this->createCartItem($name, $price, $quantity, $options);

        $content = $this->getContent();

        if ($content->has($id)) {
            $cartItem->put('quantity', $content->get($id)->get('quantity') + $quantity);
        }

        $content->put($id, $cartItem);

        $this->session->put(self::DEFAULT_INSTANCE, $content);
    }

    /**
     * Updates the quantity of a cart item.
     *
     * @param string $id
     * @param string $action
     * @return void
     */
    public function update(string $id, string $action): void
    {
        $content = $this->getContent();

        if ($content->has($id)) {
            $cartItem = $content->get($id);

            switch ($action) {
                case 'plus':
                    $cartItem->put('quantity', $content->get($id)->get('quantity') + 1);
                    break;
                case 'minus':
                    $updatedQuantity = $content->get($id)->get('quantity') - 1;

                    if ($updatedQuantity < self::MINIMUM_QUANTITY) {
                        $updatedQuantity = self::MINIMUM_QUANTITY;
                    }

                    $cartItem->put('quantity', $updatedQuantity);
                    break;
            }

            $content->put($id, $cartItem);

            $this->session->put(self::DEFAULT_INSTANCE, $content);
        }
    }

    /**
     * Removes an item from the cart.
     *
     * @param string $id
     * @return void
     */
    public function remove(string $id): void
    {
        $content = $this->getContent();

        if ($content->has($id)) {
            $this->session->put(self::DEFAULT_INSTANCE, $content->except($id));
        }
    }

    /**
     * Clears the cart.
     *
     * @return void
     */
    public function clear(): void
    {
        $this->session->forget(self::DEFAULT_INSTANCE);
    }

    /**
     * Returns the content of the cart.
     *
     * @return Illuminate\Support\Collection
     */
    public function content(): Collection
    {
        return is_null($this->session->get(self::DEFAULT_INSTANCE)) ? collect([]) : $this->session->get(self::DEFAULT_INSTANCE);
    }

    /**
     * Returns total price of the items in the cart.
     *
     * @return string
     */
    public function total(): string
    {
        $content = $this->getContent();

        $total = $content->reduce(function ($total, $item) {
            return $total += $item->get('price') * $item->get('quantity');
        });

        return number_format($total, 2);
    }

    /**
     * Returns the content of the cart.
     *
     * @return Illuminate\Support\Collection
     */
    protected function getContent(): Collection
    {
        return $this->session->has(self::DEFAULT_INSTANCE) ? $this->session->get(self::DEFAULT_INSTANCE) : collect([]);
    }

    /**
     * Creates a new cart item from given inputs.
     *
     * @param string $name
     * @param string $price
     * @param string $quantity
     * @param array $options
     * @return Illuminate\Support\Collection
     */
    protected function createCartItem(string $name, string $price, string $quantity, array $options): Collection
    {
        $price = floatval($price);
        $quantity = intval($quantity);

        if ($quantity < self::MINIMUM_QUANTITY) {
            $quantity = self::MINIMUM_QUANTITY;
        }

        return collect([
            'name' => $name,
            'price' => $price,
            'quantity' => $quantity,
            'options' => $options,
        ]);
    }
}
```

As I've already said, service classes are not something built into the framework hence there is no `artisan make` command for creating a service class. You keep the classes anywhere you like. I'm keeping my classes inside `App/Services` directory.

The `CartService.php` file contains both `public` and `protected` methods. The public methods named `add()`, `remove()`, `update()`, `clear()` are responsible for adding item to cart, removing item from cart, updating cart item quantity and clearing the cart respectively.

The other public methods are `content()` and `total()` responsible for returning the cart content and total price of added items respectively.

Finally the protected methods `getContent()` and `createCartItem()` are responsible for returning cart content within the class methods and forming a new cart item from the received parameters.

Now that the service class is ready, you need to use it inside the controller. To utilize the service class inside the `CartItemController.php` file, the code needs to be updated as follows:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Services\CartService;
use App\Http\Requests\CartItemRequest;

class CartItemController extends Controller
{
    protected $cartService;

    /**
     * Instantiate a new controller instance.
     *
     * @return void
     */
    public function __construct(CartService $cartService)
    {
        $this->cartService = $cartService;
    }

    /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
        $content = $this->cartService->content();
        $total = $this->cartService->total();

        return view('cart.index', [
            'content' => $content,
            'total' => $total,
        ]);
    }

    /**
     * Store a newly created resource in storage.
     *
     * @param  \App\Http\Requests\CartItemRequest  $request
     * @return \Illuminate\Http\Response
     */
    public function store(CartItemRequest $request)
    {
        $this->cartService->add($request->id, $request->name, $request->price, $request->quantity, $request->options);

        return back()->with('success', 'Item added to cart');
    }

    /**
     * Display the specified resource.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function show($id)
    {
        $content = $this->cartService->content();

        $item = $content->get($id);

        return view('cart', compact('item'));
    }

    /**
     * Update the specified resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function update(Request $request, $id)
    {
        $this->cartService->update($id, $request->id);

        return back()->with('success', 'Item updated in cart');
    }

    /**
     * Remove the specified resource from storage.
     *
     * @param  int  $id
     * @return \Illuminate\Http\Response
     */
    public function destroy($id)
    {
        $this->cartService->remove($id);

        return back()->with('success', 'Item removed from cart');
    }
}
```

Thanks to zero configuration resolution, the service container resolves any class that doesn't depend on any interface automatically. Hence simply injecting the `CartService` in the controller constructor does the trick:

```php
class CartItemController extends Controller
{
    protected $cartService;

    /**
     * Instantiate a new controller instance.
     *
     * @return void
     */
    public function __construct(CartService $cartService)
    {
        $this->cartService = $cartService;
    }
}
```

Now an instance of the `CartService` class becomes available within the controller and can be accessed as `$this->cartService` property. Rest of the controller action have been updated to make use of the service and as you can see, the controller has become much cleaner now.

# Action Re-usability

Apart from making the controller cleaner, you also get the benefit of the shopping cart related actions being accessible from anywhere. Consider the following livewire component for example:

```php
<?php

namespace App\Http\Livewire;

use App\Facades\Cart;
use Livewire\Component;
use Illuminate\Contracts\View\View;

class ProductComponent extends Component
{
    public $product;
    public $quantity;

    /**
     * Mounts the component on the template.
     *
     * @return void
     */
    public function mount(): void
    {
        $this->quantity = 1;
    }

    /**
     * Renders the component on the browser.
     *
     * @return \Illuminate\Contracts\View\View
     */
    public function render(): View
    {
        return view('livewire.product');
    }

    /**
     * Adds an item to cart.
     *
     * @return void
     */
    public function addToCart(): void
    {
        Cart::add($this->product->id, $this->product->name, $this->product->unit_price, $this->quantity);
        $this->emit('productAddedToCart');
    }
}
```

You can add, remove, update or clear the shopping cart from anywhere. Prior to the service class implementation, the only way to manage the cart was through HTTP requests. Now you can even manage the cart through `artisan` commands.

# Closing Thoughts

The concept of service classes discussed in this article is nothing concrete and I'm not claiming it to be a silver bullet. It's something that I've used in the past and have had no problem whatsoever. If you want to look at some other approach, consider reading about action classes [here](https://freek.dev/1371-refactoring-to-actions/).