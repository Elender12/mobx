# MOBservable

*Changes are coming!*

MOBservable is light-weight stand alone observable implementation, based on the ideas of observables in bigger frameworks like `knockout`, `ember`, but this time without 'strings attached'. Furthermore it should fit well in any typescript project.

# Observable values

The `mobservable.value(valueToObserve)` method (or just its shorthand: `mobservable(valueToObserve)`) takes a value or function and creates an observable value from it. A quick example:

```typescript
import mobservable = require('mobservable');

var vat = mobservable(0.20);

var order = {};
order.price = mobservable(10),
order.priceWithVat = mobservable(() => order.price() * (1 + vat()));

order.priceWithVat.observe((price) => console.log("New price: " + price));

order.price(20);
// Prints: New price: 24
vat(0.10);
// Prints: New price: 22
```

## mobservable.value(value, scope?):IObservableValue

Constructs a new observable value. The value can be everything that is not a function, or a function that takes no arguments and returns a value. In the body of the function, references to other properties will be tracked, and on change, the function will be re-evaluated. The returned value is an `IProperty` function/object. Passing an array or object into the `value` method will only observe the reference, not the contents of the objects itself. To observe the contents of an array, use `mobservable.array`, to observe the contents of an object, just make sure its (relevant) properties are observable values themselves.

The method optionally accepts a scope parameter, which will be returned by the setter for chaining, and which will be used as scope for calculated properties, for example:

```javascript
var value = mobservable.value;

function OrderLine(price, amount) {
	this.price = value(price);
	this.amount = value(amount);
	this.total = value(function() {
		return this.price() * this.amount();
	}, this)
}
```

## mobservable.array(initialValues?):ObservableArray

**Note: ES5 environments only**

Constructs an array like, observable structure. An observable array is a thin abstraction over native arrays that adds observable properties. The only noticable difference between built-in arrays is that these arrays cannot be sparse, that is, values assigned to an index larger than `length` are not oberved (nor any other property that is assigned to a non-numeric index). In practice, this should harldy be an issue. Example:

```javascript
var numbers = mobservable.array([1,2,3]);
var sum = mobservable.value(function() {
	return numbers.reduce(function(a, b) { return a + b }, 0);
});
sum.observe(function(s) { console.log(s); });

numbers[3] = 4;
// prints 10
numbers.push(5,6);
// prints 21
numbers.unshift(10)
// prints 31
```

## mobservable.defineProperty(object, name, value)

**Note: ES5 environments only**

Defines a property using ES5 getters and setters. This is useful in constructor functions, and allows for direct assignment / reading from observables:

```javascript
var vat = mobservable.value(0.2);

var Order = function() {
	mobservable.defineProperty(this, 'price', 20);
	mobservable.defineProperty(this, 'amount', 2);
	mobservable.defineProperty(this, 'total', function() {
		return (1+vat()) * this.price * this.amount; // price and amount are now properties!
	});
};

var order = new Order();
order.price = 10;
order.amount = 3;
// order.total now equals 36
```

In typescript, it might be more convenient for the typesystem to directly define getters and setters instead of using `mobservable.defineProperty`:

```typescript
class Order {
	_price = new mobservable.property(20, this);
	get price() {
		return this._price();
	}
	set price(value) {
		this._price(value);
	}
}
```

## mobservable.watch(func, onInvalidate)

`watch` invokes `func` and returns a tuple consisting of the return value of `func` and an unsubscriber. `watch` will track which observables `func` was observing, but it will *not* recalculate `func` if necessary, instead, it will fire the `onInvalidate` callback to notify that the output of `func` can no longer be trusted.

The `onInvalidate` function will be called only once, after that, the watch has finished. To abort a watch, use the returned unsubscriber.

`Watch` is useful in functions where you want to have a function that responds to change, but where the function is actually invoked as side effect or part of a bigger change flow or where unnecessary recalculations of `func` or either pointless or expensive, for example in React component render methods

## mobservable.batch(workerFunction)

Batch postpones the updates of computed properties until the (synchronous) `workerFunction` has completed. This is useful if you want to apply a bunch of different updates throughout your model before needing the updated computed values, for example while refreshing a value from the database.

## mobservable.onReady(listener) / mobservable.onceReady(listener)

The listener is invoked each time the complete model has become stable. The listener is always invoked asynchronously, so that even without `batch` the listener is only invoked after a bunch of changes have been applied

`onReady` returns a function with wich the listener can be unsubscribed from future events

## `IObservableValue` objects

### IObservableValue()

If an IObservableValue object is called without arguments, the current value of the observer is returned

### IObservableValue(newValue)

If an IObservable object is called with arguments, the current value is updated. All current observers will be updated as well.

### IObservableValue.observe(listener,fireImmediately=false)

Registers a new listener to change events. Listener should be a function, its first argument will be the new value, and second argument the old value.

Returns a function that upon invocation unsubscribes the listener from the property.

## `ObservableArray`

An `ObservableArray` is an array-like structure with all the typical behavior of arrays, so you can freely assign new values to (non-sparse) indexes, alter the length, call array functions like `map`, `filter`, `shift` etc. etc. All the ES5 features are in there. Additionally available methods:

### ObservableArray.clear()

Removes all elements from the array and returns the removed elements. Shorthand for `ObservableArray.splice(0)`

### ObservableArray.replace(newItemsArray)

Replaces all the items in the array with `newItemsArray`, and returns the old items.

### ObservableArray.spliceWithArray(index, deleteCount, newItemsArray)

Similar to `Array.splice`, but instead of accepting a variable amount of arguments, the third argument should be an array containing the new arguments.

### ObservableArray.observe(callback)

Register a callback that will be triggered every time the array is altered. A method to unregister the callback is returned.

In the near feature this will adhere to the ES7 specs for Array.observe so this class can be used as ES7 array shim.

### ObservableArray.values()

Returns all the values of this ObservableArray as native, non-observable, javascript array. The returned array is a shallow copy.