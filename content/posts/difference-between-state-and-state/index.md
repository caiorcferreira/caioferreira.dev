---
author: "Caio Ferreira"
title: Difference between state and State
date: "2018-12-18T12:00:00"
tags: [javascript, state machine, functional, oop]
---

![](./cover.jpg)

Photo by [Annie Spratt](https://unsplash.com/photos/hzdgFPz1V24?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/search/photos/machine?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

Today we will try to solve the ambiguity in the concept of state, highlighting the differences in the two main notions about it. The post describes the nature of each one, some use cases and how they fit in the object and functional paradigms.

# Introduction

On the last couple of months, I dove into the topic of State Machines and how we can design UI’s with this concept in order to provide better semantic and predictability to our application. When reading and talking about it often I have to stop and clarify which of the two ideas about state I am referring to:

- the idea of a collection of data at a point in time
- the idea of the representation of an entity modeled as a state machine.

For the sake of comprehension, we will use **state** to refer to the first and **State** to the last.

# Requirements

We will use Typescript for our yummy examples so some familiarity with it would be good.

# The state as a travel bag

The first notion we became comfortable with when learning about state in software development is the entity “travel bag”. Basically, we see a state as a collection of data in a specific point in time. Throughout the application lifecycle, this data is manipulated and altered in order to reflect the business process. For example:

```typescript
class Pizza {
  private dough: Dough; // it's an enum that could be traditional or thin
  private ingredients: Array<Ingredient>;

  // entity controls
  private isBeingnPrepared: boolean;
  private isBaking: boolean;
  private baked: boolean;

  constructor() {
    this.isBeingnPrepared = true;
    this.isBaking = false;
    this.baked = false;
  }

  // getters and setters

  public async bakePizza(): void {
    const oven = new OvenService();

    try {
      this.isBeingnPrepared = false;
      this.isBaking = true;

      await oven.bake(this);

      this.baked = true;
    } catch (error) {
      throw error;
    }
  }
}
```

From this point on, the pizza state, and hence the application state, has changed, because its data was updated. However, two booleans can be arranged in four different ways, and some of them are invalid states. In the object-oriented paradigm we would avoid this by encapsulating such data in an object and modeling its operations only through methods that guarantee atomic and consistent changes.

Until this use case, our model seems to be fine. But, the time comes to implement the next step in the pizzeria flow, the delivery.

```typescript
class Pizza {
  private dough: Dough;
  private ingredients: Array<Ingredient>;

  // entity controls
  private isBeingnPrepared: boolean;
  private isBaking: boolean;
  private baked: boolean;
  private isBeingDelivered: boolean;
  private hasBeenDelivered: boolean;

  constructor() {
    this.isBeingnPrepared = true;
    this.isBaking = false;
    this.baked = false;
    this.isBeingDelivered = false;
    this.hasBeenDelivered = false;
  }

  // getters and setters

  // bake behavior

  public async deliveryPizza() {
    if (!this.baked) {
      throw new PizzaNotBakedException();
    }

    const deliveryService = new DevelieryService();

    try {
      this.isBeingnPrepared = false;
      this.isBeingDelivered = true;

      await deliveryService.send(this);
    } catch (error) {
      throw error;
    }
  }

  public notifyDelivery(wasSuccessful) {
    if (wasSuccessful) {
      this.hasBeenDelivered = true;
    }
  }
}
```

What raises a flag in this code is the use of a guard condition at the start of the delivery function that checks if the pizza is baked. If not, it throws an exception. This seems really simple, and if this were the only condition, it would be fine. But, a pizza could already be left for delivery, as such, we don’t want to try to send it again. So, we add another guard condition to our function:

```typescript
class Pizza {
  private dough: Dough;
  private ingredients: Array<Ingredient>;

  // entity controls
  private isBeingPrepared: boolean;
  private isBaking: boolean;
  private baked: boolean;
  private isBeingDelivered: boolean;
  private hasBeenDelivered: boolean;

  // constructor

  // getters and setters

  // bake behavior

  public async deliveryPizza() {
    if (!this.baked) {
      throw new PizzaNotBakedException();
    }

    if (this.isBeingDelivered) {
      throw new PizzaAlreadyLeftException();
    }

    const deliveryService = new DevelieryService();

    try {
      this.isBeingPrepared = false;
      this.isBeingDelivered = true;

      await deliveryService.send(this, this.notifyDelivery);
    } catch (error) {
      throw error;
    }
  }

  // notify delivery behavior
}
```

If we elaborate all the scenarios which a pizza can be, this kind of implementation with lots of branches and conditions expressed by `if/else` statements grows exponentially. It increases our code cyclomatic complexity and diminishes maintainability as such code is more fragile, harder to read and understand.

It gets worse when this kind of conditional start to spread across the code, as in the bake function, which needs to be updated in order to not try to bake it again.

```typescript
class Pizza {
  private dough: Dough;
  private ingredients: Array<Ingredient>;

  // entity controls
  private isBeingPrepared: boolean;
  private isBaking: boolean;
  private baked: boolean;
  private isBeingDelivered: boolean;
  private hasBeenDelivered: boolean;

  // constructor

  // getters and setters

  public async bakePizza(): void {
    if (this.baked) {
      throw new PizzaAlreadyBakedException();
    }

    const oven = new OvenService();

    try {
      this.isBeingPrepared = false;
      this.isBaking = true;

      await oven.bake(this);

      this.baked = true;
    } catch (error) {
      throw error;
    }
  }

  // delivery behavior

  // notify delivery behavior
}
```

Although this kind of design serves several proposes, in special on more simple or data-centric scenarios, in fast evolution and process-centric domains it evolves on a mess of code execution paths and unsynchronized conditionals through different functions.

The state as an entity travel bag has a use and it is to carry the associated information to the model. Try to control the behavior of this entity through the same concept ends up overloading it with responsibility and creating a silent trap for our design.

The problem faced here is that the application architecture allows for invalid behavior through invalid **states**, and when it does eventually some use case will expose the bugs created by this freedom. Besides that, this approach takes the system invariants, in this case, the Pizza cooking flow, and scatter then inside many implementation points instead of enforcing them in the design.

_Side note: if you are versed in Algebraic Data Types you can see this as a Product Type with cardinality which tends to infinity._

# Representational State

Once we have the problem of control the entity information and behavior being done by the same construct, the **state**, our response could not be more simple: let’s break these responsibilities.

Therefore, we need a new pattern to handle our entity’s behavior.

But, the alternative pattern we propose when designing your application is not at all new. It is the State Pattern, describes in many ancient books about OO. And this books will tell you the same, that the State Pattern seeks to delegate an entity behavior to a specific implementation which is the current State and at the end of the method calculate the entity’s next State, which will now represent the entity, replacing its behaviors implementation on the fly. After all, this pattern is a translation of a state machine to the _idiom of the nouns._ An alternative implementation for our Pizza example can be as below:

```typescript
interface IPizza {
  bakePizza();
  deliveryPizza();
  notifyDelivery(wasSuccessful: boolean);
}

type Pizza = PreparingPizza | BakedPizza | DeliveringPizza | DeliveredPizza;

class PreparingPizza implements IPizza {
  private dough: Dough; // it is an enum that could be traditional or thin
  private ingredients: Array<Ingredient>;

  constructor(dough: Dough, ingredients: Array<Ingredient>) {
    this.dough = dough;
    this.ingredients = ingredients;
  }

  // setters

  getDough() {
    return this.dough;
  }

  getIngredients() {
    return this.ingredients;
  }

  public async bakePizza(): Promise<BakedPizza> {
    const oven = new OvenService();

    try {
      await oven.bake(this);

      return new BakedPizza(this);
    } catch (error) {
      throw error;
    }
  }

  public async deliveryPizza() {
    throw new PizzaNotReadyForDelivery();
  }

  public notifyDelivery(wasSuccessful) {
    throw new PizzaNotReadyForDelivery();
  }
}

class BakedPizza implements IPizza {
  private dough: Dough;
  private ingredients: Array<Ingredient>;

  // constructor
  constructor(pizza: PreparingPizza) {
    this.dough = pizza.getDough();
    this.ingredients = pizza.getIngredients();
  }

  // getters and setters

  public async bakePizza(): Promise<BakedPizza> {
    throw new PizzaAlreadyBakedException();
  }

  public async deliveryPizza(): Promise<DeliveringPizza> {
    const deliveryService = new DevelieryService();

    try {
      await deliveryService.send(this);

      return new DeliveringPizza(this);
    } catch (error) {
      throw error;
    }
  }

  public notifyDelivery(wasSuccessful) {
    throw new PizzaNotLeftForDeliveryYey();
  }
}

class DeliveringPizza implements IPizza {
  private dough: Dough;
  private ingredients: Array<Ingredient>;

  // constructor

  // getters and setters

  public async bakePizza(): Promise<BakedPizza> {
    throw new PizzaAlreadyBakedException();
  }

  public async deliveryPizza(): Promise<DeliveringPizza> {
    throw new PizzaAlreadyLeftForDeliveryException();
  }

  public notifyDelivery(wasSuccessful) {
    if (wasSuccessful) {
      return new DeliveredPizza(this);
    }
  }
}

class DeliveredPizza implements IPizza {
  private dough: Dough;
  private ingredients: Array<Ingredient>;

  // constructor

  // getters and setters

  public async bakePizza(): Promise<BakedPizza> {
    throw new PizzaAlreadyBakedException();
  }

  public async deliveryPizza(): Promise<DeliveringPizza> {
    throw new PizzaAlreadyLeftForDeliveryException();
  }

  public notifyDelivery(wasSuccessful) {
    throw new PizzaAlreadyDeliveredException();
  }
}
```

With this implementation, we enforce the domain invariants with our type system through the interface and the Pizza union type. With it we gain less cyclomatic complexity since we don’t have so many branches in our code and, by design, we don’t allow for invalid States to happen. Besides that, each State carries an internal data, its _travel bag_. As such, these patterns are not excluding, but rather composable.

In the trend on the front-end what we are usually seeing is more a functional paradigm approach to the state machines. The entity, represented as a state machine, is now just a different data structure for each State that can be interpreted by the pure functions that implement the domain behaviors. These functions than can internally delegate its call to others functions specialized in each State. This separation of the state machine implementation of the behavior is natural as it follows the idiom for functional architectures.

What remains in both cases are the nature of the State as an **entity’s representation**. It works on its behalf and delimits the possible behaviors it can expose.

For example, a Pizza could never be in Baked and Delivered States at the same time. Now, it isn’t an implementation that guarantees that it is the design itself. Such abstractions, that models the domain, the heart of our product, couldn’t depend on implementation details to be valid, they must depend on the abstractions itself.

_Side note: if you are versed in Algebraic Data Types you can see this as a Union Type with finite cardinality in the order of less then a dozen._

# Evolving the abstraction

One could implement a State oriented design by using a simple enum, a proper state machine implementation or a more advanced concept, a statechart.

It is true that many domains can be modeled using the two first approaches to code a State, but sometimes we are faced with a high complexity scenario where this abstraction implementation would not scale with the development of the application.

For that reason that in 1987 David Harel proposed a new technique that expanded the grounds of the state machine definition, introducing tools like state hierarchy, parallelism, clustering, history, etc. He called it statecharts and it is a formalism that helps us scale the development of a State design, be implementing it thoroughly or just taking some tools.

I highly recommend reading more about statecharts as it can shift your mindset about how to approach problems.

# Summary

Now we can differentiate **state** from **State** and avoid accidental complexity by using the right construct to model our domain. Its worth nothing if I don’t say that there is no silver bullet and these are tools to deliver a job. We have been experimenting with this design style on my team and it has been helpful since our scenario is really complex and fast pacing.

If you have any questions or want to discuss these and other topics more in deep please comment or you can reach me at Twitter, @caiorcferreira.

Thanks for reading!

# References

- [State Design Pattern](https://en.wikipedia.org/wiki/State_pattern)
- [State Driven Development for User Interfaces](https://dev.to/nimmo/state-driven-development-for-user-interfaces-part-1-an-introduction-27f1)
- [A Statechart implementation on JS](https://xstate.js.org/docs/)
- [Statecharts: A Visual Formalism For Complex Systems](http://www.inf.ed.ac.uk/teaching/courses/seoc/2005_2006/resources/statecharts.pdf) - the original article
- [Pure UI Control](https://medium.com/@asolove/pure-ui-control-ac8d1be97a8d)
- [Constructing User Interfaces with Statecharts](<https://www.amazon.com/Constructing-User-Interface-Statecharts-Horrocks/dp/0201342782%5D(https://www.amazon.com/Constructing-User-Interface-Statecharts-Horrocks/dp/0201342782)>)
