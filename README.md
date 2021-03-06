# Oodux

Oodux is an object-oriented Redux toolkit. It provides an OO wrapper around Redux, and is meant to reduce Redux setup time by eliminating the need for the user to define action types, action creators, and in many cases mapDispatchToProps functions in react-redux. The user needs only to define reducers, their initial states, and any async actions, and Oodux will do the rest. Oodux also automatically provides simple and predictable action types and creators for state properties based on their types, and allows for the use of automatically memoized getters to keep data manipulation logic out of UI components. Check out the walkthrough below.

## Example

This shows a simple example of how to use Oodux, using a single reducer. Oodux also supports multiple reducers -- see below.

### Initial State

First, create a class which extends Oodux. Instances of this class will serve as the state representations held in the store by Redux, so its constructor should create your initial/default state.

```js
import Oodux from "oodux"

class State extends Oodux {
	constructor() {
		super()
		this.user = {}
		this.pets = []
		this.nightTheme = false
		this.error = null
		this.current = null
	}

```

### Reducer

Next, create instance methods for your class. Each instance method should correspond to a "case" in your reducer. So if your Redux reducer includes a line like this:

```js
case ADD_PRODUCT:
	return {
		...state,
		products: [...state.products, action.product],
		coupons: [...state.coupons, action.coupon],
	}
```

...your Oodux class should have an instance method like the following:

```js

addProduct({product, coupon}){
	return this.add({products: product, coupons: coupon})
}

```

This method will be called on the current state instance by the reducer (which is automatically created by Oodux). So `this` will represent the current state, and `this.add` is a built-in method that creates a new state instance, copies the current instance, and then modifies it, allowing for simple and quick immutable editing.

First of all, when an instance method is defined in a Oodux subclass, upon initializing the store, Oodux will also automatically create an action type, an action creator for that type, and a static method (by the same name as the instance method) which dispatches that action created by that creator.

The action type will be equal to the name of the method. Any instance method with no arguments will yield an action creator of the form `() => ({ type })`. Any instance method with one argument will yield an action creator of the form `data => ({ type, data })`. Instance methods thus can only take zero or one arguments; use destructuring for multiple arguments.

So by creating the instance method seen above, you also get the following for free:

```js
State.__creators.addProduct // === data => ({type:"addProduct", data})
State.addProduct // === data => dispatch(State.creators.addProduct(data))
```

Oodux also creates a reducer for you which takes in the action, checks whether there is an instance method by the same name, and if there is, invokes it on the action's payload. If no method matches the action's type, it returns the current state; there is no need to write a method to handle default cases.

### Overriding Default Behavior

If you want to create a utility instance method but you do not want Oodux to turn it into an action or use it in the reducer, simply prefix it with an underscore.

```js

	_pickRandomPet(){
		const idx = Math.floor(Math.random() * this.pets.length)
		return this.pets[idx]
	}

	selectRandomPet(){
		return this.setCurrent(this._pickRandomPet())
	}

```

### Immutable Editing Helpers

Oodux comes with a handful of immutable editing helper functions, like `this.add` seen above. Oodux will also automatically create magic instance methods (with corresponding action types, creators, and static dispatching methods) for each property on your initial state, depending on the type of its initial value; `this.setCurrent` seen above is an example of that; it is automatically created because the initial state includes a property called `current`. See the API for details.

### Async Actions

There is no need to use thunk middleware. Async actions should be static methods on your Oodux subclass. And because Oodux subclasses have direct access to the redux store, there is no need to abstract over the dispatch or getState functions.

```js

	static async login(email, password) {
		try {
			const { data } = await axios.post("/auth/login", {email, password})
			this.setUser(data)
		} catch (error) {
			this.setError(error)
		}
	}

	static async logout() {
		try {
			await axios.post("/auth/logout")
			this.removeUser()
		} catch (error) {
			this.setError(error)
		}
	}

	static async getPets() {
		try {
			const { id } = this.getState().user
			const { data } = await axios.get(`/api/users/${id}/pets`)
			this.setPets(data)
		} catch (error) {
			this.setError(error)
		}
	}

```

These methods should simply implement the async logic you need, then dispatch directly to the store by calling the static methods which correspond to either the instance methods you created for the reducer or magic methods created automatically. Note that because these are static methods, `this` refers to the class, not an instance -- so `this.setPets(data)` above is not actually an invocation of the instance method that the user created -- it is invoking the static method created automatically by Oodux.

Note that the current state is also accessible from your Oodux subclass by calling the `getState` static method as in the `getPets` static method above.

If you are using multiple reducers, and wish to keep your async actions in a separate file from your reducers, you can put them on the parent class; see Multiple Reducers below.

### React-Redux

When using React-Redux, Oodux often eliminates the need to use mapDispatchToProps.

```js
import React from "react"
import { connect } from "react-redux"
import State from "./example/store"

const DeletePetsView = ({ user, pets }) => (
	<div>
		<h1>{`${user.name}'s Pets: Click to Delete`}</h1>
		<ul>
			{pets.map(pet => (
				<li onClick={() => State.deletePet(pet)}>{pet.name}</li>
			))}
		</ul>

		<button onClick={() => State.toggleView()}>Toggle View</button>
	</div>
)

const mapStateToProps = state => ({ user: state.user, pets: state.pets })

export default connect(mapStateToProps)(DeletePetsView)
```

Note that instead of importing the whole State class, we could also add lines to our State module to export just those async methods. This is made easy by the fact that automatically created static methods are always bound.

```js
export const { deletePet, toggleView } = State
```

When creating custom static async methods, it suggested that these should use static field and arrow notation to allow for easy export without the need to manually bind.

```js
//store.js
export default class State extends Oodux {
	static fetchData = async (data) => {
		//...
	}
}

export const {fetchData} = State

//file2

import {fetchData} from "./store.js"

export default props => <button onClick={() => fetchData(props.id)}>

```

### Memoized Getters

Getters defined in the body of your Oodux subclass will not be made into action creators, no matter how you name them. But they can be useful for allowing front end components to subscribe to processed data without duplicating information in your store or putting data processing logic in your UI components.

Whereas without Oodux you may have a mapStateToProps that looks like this:

```js
const mapStateToProps = state => ({
	user: state.user,
	pets: state.pets,
	highestRankedPet: state.pets.reduce((bestPet, currentPet) =>
		currentPet.rank > bestPet.rank ? currentPet : bestPet
	),
})
```

Oodux allows for better seperation of concerns, having the following method in your Oodux subclass:

```js
get highestRankedPet(){
	return this.pets.reduce((bestPet, currentPet) =>
		currentPet.rank > bestPet.rank ? currentPet : bestPet
	)
}
```

With this getter defined, a front-end component could subscribe to `state.highestRankedPet` using mapStateToProps, and avoid calculating the `highestRankedPet` within that component:

```js
const mapStateToProps = state => ({
	user: state.user,
	pets: state.pets,
	highestRankedPet: state.highestRankedPet,
	),
})
```

Any getters defined on your Oodux subclass will also be automatically memoized, so that it is only re-run when the state components that it derives its value from change. For example, in the case of `get highestRankedPet()` above, this value will be cached and returned without recalculating until the value of `this.pets` changes. Note that if your getter includes conditional logic that depends on non-state information to determine which state properties it accesses, the memoization will not work properly. To prevent a getter from being automatically memoized, prefix it with `_`.

### Initializing

Instead of invoking the Redux function `createStore`, call the `init` method on your Oodux subclass; this returns the Redux store.

```js
export default State.init()
```

To use any Redux Middleware, simply call the `applyMiddleware` static method on your Oodux class. Here is an example with a single reducer.

```js
export default State.applyMiddleware(createLogger({ collapsed: true })).init()
```

To use a Redux Middleware which wraps the Redux `applyMiddleware` function, simply call the `wrapMiddleware` static method on your Oodux (sub)class. Here is an example with multiple reducers.

```js
export default State.wrapMiddleware(reduxDevTools).init()
```

### Multiple Reducers

Oodux supports the multiple reducers pattern seen in Redux. In order to do this, first determine a parent class -- it can be the class exported by the Oodux package, or you can create a subclass which extends Oodux. Then create classes which extend the parent class -- one for each slice of your state. Then call the `initSlices` method on the parent class, passing in its subclasses as arguments.

```js
export default Oodux.initSlices(Users, Pets, Parks)
```

This will create a state object whose keys are camel-cased versions of the names of the relevant subclasses. Note that parent class is not included in the state object, so there is no point in defining a constructor on the parent class.

It may still be worthwhile to define a custom parent class instead of using Oodux for the following reasons: 1) you may wish to create utility methods which are available to all subclasses (note that these do not need to be prefixed with an underscore, because Oodux will not try to make action creators for methods on the parent class); 2) you may wish to put some or all of your async actions on your parent class as a way to organize your redux logic.

The static method `getSlice()` returns the current state of the slice corresponding to the calling class.

If you want to make multiple reducers react to the same action type just give each class an instance method by the same name.

```js
class User extends Oodux {
	...
	removeUser(){
		return new User()
	}
}

class Cart extends Oodux {
	...
	removeUser(){
		return new Cart()
	}
}

```

In the above example, we want to clear out our application state when the user logs out, so when the action `"removeUser"` is dispatched, both of the above reducing methods are invoked, returning a state whose `user` and `cart` slices have been returned to their initial setting.

By default, all automatically created actions and dispatching methods are attached to the parent class so that they are accessible from any child class. This means, for instance, that an async action defined in one class can dispatch actions defined in any of the other classes. Automatically created magic reducing methods are attached to the prototype of the relevant subclass, but their corresponding action creators and dispatchers go to the parent class.

Note that if two or more state slices have properties of the same name, the magic reducing methods will still be created, but no action creators or disaptchers will be created.

## API

### Static Properties

#### Oodux.\_\_creators

An object whose keys are the action types, and whose values are the corresponding action creators. Automatically populated by `Oodux.initSlices` or `Oodux.init`.

#### Oodux.store

The redux store. When using multiple reducers, each Oodux subclass has a reference to the store, but there is still only one store.

### Initializing Static Methods

#### Oodux.applyMiddleware(middleware1[, ...[, middlewareN]])

Stores the provided middlewares on `Oodux.middlewares`, which are then passed into Redux's `applyMiddleware` by `Oodux.init` or `Oodux.initSlices`. Returns the invoking Oodux (sub)class. The chaining order with `Oodux.wrapMiddleware` does not matter.

```js
State.applyMiddleware(mware).init()
State.wrapMiddleware(reduxDevTools).applyMiddleware(mware).init()
State.applyMiddleware(mware).wrapMiddleware(reduxDevTools).init()
Oodux.applyMiddleware(mware).initSlices(User, Products, Orders)
```

#### Oodux.initSlices(OoduxSubclass1, OoduxSubclass2[, ...[, OoduxSubclassN]])

Creates and combines the reducers on each Oodux subclass, applies and wraps any middleware passed in by `Oodux.applyMiddleware` and `Oodux.wrapMiddleware`, and creates and returns the store. This method also creates the action types, action creators, and dispatching static methods to correspond to each instance method on the OoduxSubclasses, as well as magic methods and their corresponding action types, creators, and dispatchers based on the initial state of each class, and memoizes any getters on each subclass. The dispatching static methods are attached to the invoking class of this method, which makes all static dispatchers available to all subclasses. When calling this method, do not call `Oodux.init`.

```js
Oodux.initSlices(Dogs, Cats, PetStores)
```

#### Oodux.init()

Creates the reducer, applies and wraps any middleware passed in by `Oodux.applyMiddleware` and `Oodux.wrapMiddleware`, and creates and returns the store. This method also creates the action types, action creators, and dispatching static methods to correspond to each instance method on the invoking Oodux subclass, as well as magic methods and their corresponding action types, creators, and dispatchers based on the initial state of the class, and memoizes any getters. When calling this method, do not call `Oodux.initSlices`.

```js
export default State.applyMiddleware(createLogger({ collapsed: true })).init()
```

#### Oodux.wrapMiddleware(middleware1[, ...[, middlewareN]])

Stores the provided middlewares on `Oodux.wrappingMiddlewares`, which are then composed (in the order passed in) with Redux's `applyMiddleware` by `Oodux.init` or `Oodux.initSlices`. Returns the invoking Oodux (sub)class. The chaining order with `Oodux.applyMiddleware` does not matter.

```js
State.wrapMiddleware(reduxDevTools).init()
State.wrapMiddleware(reduxDevTools).applyMiddleware(mware).init()
State.applyMiddleware(mware).wrapMiddleware(reduxDevTools).init()
```

### Other Static Methods

#### Oodux.getState()

Returns the current state. Just a shorthand for `Oodux.store.getState()`.

```js
const currentState = this.getState()
const { students } = this.getState()
const { id } = this.getState().user
```

#### Oodux.getSlice()

Returns the current state for the calling class's slice. If not using multiple reducers, this is equivalent to `getState`.

```js
class Pets extends Oodux {
	static async getPets() {
		try {
			const { favorites } = this.getSlice() //same as this.getState().pets
			const { data } = await axios.get(`/api/pets/${favorites[0].id}`)
			this.setPets(data)
		} catch (error) {
			this.setError(error)
		}
	}
}
```

#### Oodux.from(obj)

Creates a new state object by calling the invoking class's constructor, then copies the values of any keys shared by the state object and `obj` onto the state object. Note that if `obj` has keys which do not exist on the state object, they will not be copied onto the new state object. Equivalent to calling `new Oodux().update(obj)`.

```js
loadNewUser(dataFromAxios){
	return State.from(dataFromAxios)
}

const state = new State() //state === {color: "white", thickness: 0, shape: null}
const wideBrush = {thickness: 4, shape: "circle", created: "2020-06-02T19:04:39.865Z"}
const newState = State.from(wideBrush) //newState === {color: "white", thickness: 4, shape: "circle"}
```

### Instance Methods

#### Oodux.prototype.add(obj)

Creates `newState`, a copy of the invoking instance, then for each `key` in object, pushes the corresponding value onto `newState[key]`, then returns `newState`. When adding only to a single array on state, you may prefer to use the magic method for that array.

```js
addNewProduct(product){
	return this.add({products: product, newProductIds: product.id})
}
```

#### Oodux.prototype.copy()

Returns a shallow copy of the invoking instance. This method is called by most other provided instance methods.

```js
rotateDirections(){
	const newState = this.copy();
	[newState.north, newState.east, newState.south, newState.west] = [newState.east, newState.south, newState.west, newState.north];
	return newState;
}
```

#### Oodux.prototype.remove(obj)

Creates `newState`, a copy of the invoking instance, then for each `key` in object, removes the corresponding value from `newState[key]` (if it is present), then returns `newState`. When removing only from a single array on state, you may prefer to use the magic method for that array.

```js
removeProduct(product){
	return this.remove({products: product, bestSellerIds: product.id })
}
```

#### Oodux.prototype.removeById(obj)

Works just like `Oodux.prototype.remove` except that values on the input object are expected to be ids; removes the first element in each array that has the corresponding id.

```js
removeProduct(product){
	return this.removeById({products: product.id, coupons: product.coupon.id})
}
```

#### Oodux.prototype.update(obj)

Creates a copy of the invoking instance, and then for all keys shared by the invoking instance and `obj`, the value of `obj` at that key is copied onto the new copy. Note that if `obj` has keys which do not exist on the state object, they will not be copied onto the new state object.

```js
loadUser(userFromAxios){
	return this.update(userFromAxios)
}
```

### Magic Methods

These methods are automatically created for every property on the initial state as defined in your Oodux subclass's constructor.

#### Oodux.prototype.set{PropertyName}(value)

#### Oodux.set{PropertyName}(value)

Creates a copy of the invoking instance and updates `copy[PropertyName]` to have value `value`, and then returns the copy. This method is created for all properties on the state object.

The corresponding static method dispatches the action called `set{PropertyName}`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.count = 0
	}
}

State.init()

State.setCount(42) // this.store.getState().count === 42
```

#### Oodux.prototype.increment{NumericPropertyName}(value)

#### Oodux.increment{NumericPropertyName}(value)

Creates a copy of the invoking instance and updates `copy[NumericPropertyName]` to be `copy[NumericPropertyName] + value`, and then returns the copy. This method is created for all properties on the state object whose initial value is a number.

The corresponding static method dispatches the action called `increment{NumericPropertyName}`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.count = 21
	}
}

State.init()

State.incrementCount(21) // this.store.getState().count === 42
```

#### Oodux.prototype.toggle{BooleanPropertyName}()

#### Oodux.toggle{BooleanPropertyName}()

Creates a copy of the invoking instance and updates `copy[BooleanPropertyName]` to be `!copy[BooleanPropertyName]`, and then returns the copy. This method is created for all properties on the state object whose initial value is a boolean.

The corresponding static method dispatches the action called `toggle{BooleanPropertyName}`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.data = []
		this.error = false
	}
}

State.init()

State.toggleError() // this.store.getState().error === true
```

#### Oodux.prototype.addTo{ArrayPropertyName}(value1[, ...[, valueN]])

#### Oodux.addTo{ArrayPropertyName}(value1[, ...[, valueN]])

Creates a copy of the invoking instance and updates `copy[ArrayPropertyName]` to be `copy[ArrayPropertyName].concat([value1, ..., valueN])`, and then returns the copy. This method is created for all properties on the state object whose initial value is an array.

The corresponding static method dispatches the action called `addTo{ArrayPropertyName}`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.data = [42]
		this.error = false
	}
}

State.init()

State.addToData(34, 47, 52) // this.store.getState().data === [42, 34, 47, 52]
```

#### Oodux.prototype.removeFrom{ArrayPropertyName}(data)

#### Oodux.removeFrom{ArrayPropertyName}(data)

Creates a copy of the invoking instance, with `data` removed from `copy[ArrayPropertyName]`. If `data` was not originally present the copy is identical. See also `Oodux.prototype.removeFrom{ArrayPropertyName}ById`.

The corresponding static method dispatches the action called `removeFrom{ArrayPropertyName}`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.data = []
		this.error = false
	}
}

State.init()

State.setData([
	{ id: 1, name: "bulbasaur" },
	{ id: 7, name: "squirtle" },
	{ id: 4, name: "charmander" },
])
//this.store.getState().data === [{id:1, name:"bulbasaur"}, {id:7, name:"squirtle"}, {id:4, name:"charmander"}]
State.removeFromData(State.getState().data[1]) //this.store.getState().data === [{id:7, name:"squirtle"}, {id:4, name:"charmander"}]
```

#### Oodux.prototype.removeFrom{ArrayPropertyName}ById(id)

#### Oodux.removeFrom{ArrayPropertyName}ById(id)

Creates a copy of the invoking instance, with the first element having id `id` removed from `copy[ArrayPropertyName]`. If no element has id `id` the copy is identical. See also `Oodux.prototype.removeFrom{ArrayPropertyName}`.

The corresponding static method dispatches the action called `removeFrom{ArrayPropertyName}ById`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.data = []
		this.error = false
	}
}

State.init()

State.setData([
	{ id: 1, name: "bulbasaur" },
	{ id: 7, name: "squirtle" },
	{ id: 4, name: "charmander" },
])
//this.store.getState().data === [{id:1, name:"bulbasaur"}, {id:7, name:"squirtle"}, {id:4, name:"charmander"}]
State.removeFromDataById(1) //this.store.getState().data === [{id:7, name:"squirtle"}, {id:4, name:"charmander"}]
```

#### Oodux.prototype.update{ArrayPropertyName}({key, data})

#### Oodux.update{ArrayPropertyName}({key, data})

Creates a copy of the invoking instance, with the array `copy[ArrayPropertyName]` modified so that `data` is swapped in for one of its elements. The element swapped out will be the first one in the array whose value at `key` is the same as that of `data`. If the key you wish to use is `"id"`, use `Oodux.prototype.update{ArrayPropertyName}ById`.

<!-- and swaps `data` with the first element `el` of `copy[ArrayPropertyName]` such that `el[key] === data[key` and then returns the copy. This method is created for all properties on the state object whose initial value is an array. -->

The corresponding static method dispatches the action called `update{ArrayPropertyName}`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.data = []
		this.error = false
	}
}

State.init()

State.setData([
	{ id: 1, name: "bulbasaur" },
	{ id: 7, name: "squirtle" },
	{ id: 4, name: "charmander" },
])
//this.store.getState().data === [{id:1, name:"bulbasaur"}, {id:7, name:"squirtle"}, {id:4, name:"charmander"}]
State.updateData("name", { id: 5, name: "charmander" }) //this.store.getState().data === [{id:1, name:"bulbasaur"}, {id:7, name:"squirtle"}, {id:5, name:"charmander"}]
```

#### Oodux.prototype.update{ArrayPropertyName}ById(data)

#### Oodux.update{ArrayPropertyName}ById(data)

Creates a copy of the invoking instance, with the array `copy[ArrayPropertyName]` modified so that `data` is swapped in for one of its elements. The element swapped out will be the first one in the array whose id is the same as that of `data`. If you wish to swap based on a key other than `"id"`, use `Oodux.prototype.update{ArrayPropertyName}`.

The corresponding static method dispatches the action called `update{ArrayPropertyName}ById`, which in turns calls the instance method on the current state object, and sets its return value as the new (slice of) state.

```js
class State extends Oodux {
	constructor() {
		this.data = []
		this.error = false
	}
}

State.init()

State.setData([
	{ id: 1, name: "ivysaur" },
	{ id: 7, name: "squirtle" },
	{ id: 4, name: "charmander" },
])
//this.store.getState().data === [{id:1, name:"ivysaur"}, {id:7, name:"squirtle"}, {id:4, name:"charmander"}]
State.updateDataById({ id: 1, name: "bulbasaur" }) //this.store.getState().data === [{id:1, name:"bulbasaur"}, {id:7, name:"squirtle"}, {id:5, name:"charmander"}]
```
