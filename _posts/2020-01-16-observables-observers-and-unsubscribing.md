---
layout: post
title: "Observables, observers and unsubscribing"
date: 2020-01-16 10:45:00 +0100
tags: [angular, rxjs, observable, observer, unsubscribe]
author: wouterth
---

Observables are the basic building block of RxJS and Angular. They emit a stream of events to which you can subscribe. Over the course of an observable lifetime, it can emit multiple values. 

> The code samples used in this post are Angular specific, but the general principles apply to all RxJs implementations.

## The default: cold observables

Observables in RxJS are **cold** or **unicasting** by default. Each subscription to a cold observable creates their own underlying data producer. A good example of this is a HTTP call.

```typescript
@Component({ template: '', selector:'my-component'})
export class MyComponent implements OnInit {
    constructor(private http: HttpClient) { }

    ngOnInit(): void {
        this.http.get('http://mydomain.com/products').subscribe();
    }
}
```

Each time `MyComponent` is initialized in the above example, it will subscribe to the same url and perform a new HTTP call. Usually you will encapsulate the data retrieval logic in a service to make it easier to reuse.

```typescript
@Injectable({ providedIn: 'root' })
export class MyService {
    constructor(private http: HttpClient) { }

    getProducts() {
        return this.http.get('http://mydomain.com/products');
    }
}

@Component({ template: '', selector:'my-component'})
export class MyComponent implements OnInit {
    constructor(private myService: MyService) { }

    ngOnInit(): void {
        this.myService.getProducts().subscribe();
    }
}
```

Now any component subscribing to the `getProducts()` method on the `MyService` class will each trigger their own HTTP call as the data producer (in this case the HTTP call) will be created each time a subscribe is invoked.

> Always subscribe in components, never in services, as components have a lifecycle hook to do clean-up while a service is supposed to be long-lived.

## Observers

A subscription can have one parameter, which is an object called an observer. It has three properties: `next`, `complete` and `error`. They allow you to take some action based on the emission of the observable. The `getProducts()` call above makes no sense if we don't do anything with the data.

```typescript
 ngOnInit(): void {
        this.myService.getProducts().subscribe({
            next: (result) => {
                console.log('We retrieved the following data from http:', result);
            },
            error: (err) => {
                console.log('I\'m sorry, we encountered an error:', err);
            },
            complete: () => {
                console.log('Our HTTP call was completed.');
            }
        });
    }
```

The observer gives us access to our results and allows us to catch errors. The `complete` method is invoked when the observable is completed.


## Unsubscribing

Some observables -like our HTTP call- create the requested data using their data producer, emit it and then complete. Others may emit a steady stream of values. If this is the case we need to unsubscribe or the subscription will stay open forever, causing memory leaks over time. Unsubscribing can be done in multiple ways. 

### Subscriptions
The first way is to just register the subscription.

```typescript
@Component({ template: '', selector:'my-component'})
export class MyComponent implements OnInit, OnDestroy {
    private _subscription: Subscription;

    constructor(private myService: MyService) { }

    ngOnDestroy(): void {
        this._subscription.unsubscribe();
    }

    ngOnInit(): void {
        this._subscription = this.myService.getProducts().subscribe();
    }
}
```

This method works fine, but it has some drawbacks. One is that you have to keep track of the state of your subscriptions, which is a bit against the [reactive programming paradigm](https://en.wikipedia.org/wiki/Reactive_programming). Another is that for each subscription you need a new variable and remember to call it in the `OnDestroy` lifecycle hook. 

### The `takeUntil` operator

So we have the `takeUntil` operator. We subscribe to an observable until another observable (in this case a `Subject` we created ourselves) emits something.

```typescript
@Component({ template: '', selector:'my-component'})
export class MyComponent implements OnInit, OnDestroy {
    private _ngDestroyed = new Subject<void>();

    constructor(private myService: MyService) { }

    ngOnDestroy(): void {
        this._ngDestroyed.next();
        this._ngDestroyed.complete();
    }

    ngOnInit(): void {
        this.myService.getProducts().pipe(
            takeUntil(this._ngDestroyed)
        ).subscribe();
    }
}
```

Subjects and operators will be covered more in depth later in another post, suffice to know for now that the `_ngDestroyed` subject is also an observable where we control the data producer. When the `ngOnDestroy` component lifecycle hook is triggered, our subject emits and completes, cleaning up after itself. Observers listening to this subject will thus receive an emission when the component is destroyed. Because the `getProducts()` call uses the `takeUntil` operator with the `_ngDestroyed` subject it will thus keep listening until our component is destroyed. The big advantage is that we can use the same subject to notify multiple observables we want to stop listening to them, and we don't need to track the state of their individual subscriptions.

The observable part of the `_ngDestroyed` subject is called a **hot** or **multicasting** observable, where the data producer is created outside of the subscription. The values are thus shared amongst the subscribers in contrast to a **cold** observable like our HTTP call, where each subscription gets it's own instance of the data.

> Hint: when in doubt, always unsubscribe. The `takeUntil` operator will do nothing when the first observable completes. Not unsubscribing to an observable that doesn't complete itself might cause memory leaks over time as we will continue to listen to the observable even after our component is destroyed. Revisiting the component will create a new instance and thus a new subscription.

### The `async` pipe

The final way is to use the `async` pipe. [Pipes](https://angular.io/guide/pipes) are a built in Angular type to transform data inside a component template. The `async` pipe has the advantage of not needing to unsubscribe, as it's automatically destroyed when the component is destroyed. It is thus the preferred way of using observables in Angular.

```typescript
@Component({ 
    template: `
    <ng-container *ngIf="products$ | async as products">
        <product-display *ngFor="let product of products" [data]="product"></product-display>
    </ng-container>
`, 
selector:'my-component'
})
export class MyComponent implements OnInit {
    products$: Observable<any>;

    constructor(private myService: MyService) { }

    ngOnInit(): void {
        this.products$ = this.myService.getProducts();
    }
}
```

As we can see in the code snippet, the heavy lifting is done inside the template. We declare the `products$` variable in the component and assign it on initialization from the injected servie. The `products$ | async` line in our `ngIf` directive will trigger the subscription. Then the `ngFor` directive is used to loop through our results (assuming our function returns a collection). 

The async pipe not only has the cleanest syntax but also the least boilerplate code, so it should always be the preferred method for subscribing to observables.

> Note: observable names often end in a \$, this is an unofficial naming convention that is widely used. This makes it easy to spot observables immediatly and allows you to store the result of the observable in a variable with the same name without the trailing \$.

## TL;DR

For those searching for the TL;DR version:
* Most observables are by default **cold** or **unicast**, meaning the data producer is created inside the subscription.
* When created by a `Subject`, the data producer is created outside of the subscription and all subscribers will share the emitted values. This is called a **hot** or **multicasting** observable.
* Observables need to be unsubscribed or they will go on forever, most cold observables will complete themselves though (check the underlying implementation to be certain).
* It can't hurt to unsubscribe even when it's unnecessary, in contrast is does hurt **NOT** to unsubscribe when you need to. Unsubscribing is therefore considered a safe default.
* Don't subscribe inside services, only in components as components will be destroyed when unloaded providing an opportunity to unsubscribe.
* Prefer subsribing through the `async` pipe first. If this is not possible for some reason, use the `takeUntil` operator to link the subscription to the component `ngOnDestroy` lifecycle hook.

## Summary

This covers the basic of observables. Observables are a basic RxJS concept that are used in different frameworks, although in this post I focused on the Angular implementation.

If you made it this far, thanks for reading!
