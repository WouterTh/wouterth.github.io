---
layout: post
title: "Operators"
date: 2020-02-26 11:45:00 +0100
tags: [angular, rxjs, operator, observable]
author: wouterth
---

[Retrieving data from observables]({% post_url 2020-01-16-observables-observers-and-unsubscribing %}) is usually only one piece of the puzzle. More than likely the retrieved data has to be manipulated or transformed to be usable in an application. That's where operators come in.

> The code samples used in this post are Angular specific, but the general principles apply to all RxJs implementations.

## <a name="pipeable-vs-creation-operators"></a>Pipeable vs creation operators

Simply put, all operators are either pipeable operators or creation operators. A pipeable operator is a pure function that takes an observable as input and produces a different observable as output, never modifying the source observable. In this way different operations can be performed on a source observable and the result can be adapted to suit your needs. Creation operators on the other hand are not pipeable. They don't accept an observable as input, but they do produce one as output. 

## <a name="filtering-operators"></a>Filtering operators

When subscribing to an observable, filtering only on the values that are interesting for the current application state can be a powerful tool to reduce the number of executions of certain code. This becomes especially interesting for multicasting observables where we are only interested in a slice of the data.

Let's say we have an object `User` defined as follows:

```typescript
interface User {
    name: string;
    password: string;
    email: string;
}
```

And we have a `UserNameDisplayComponent` that is only concerned with displaying the name of the user. For ease of reference we'll assume the `UserService` handles the retrieval of the current user and the `getUser()` method returns an observable containing a valid `User` object.

```typescript
@Component({ 
    template: `{% raw %}    
        <p *ngIf="user$ | async as user">Welcome, {{user?.name}}</p>
        {% endraw %}`, 
    selector:'user-name-display-component'
})
export class UserNameDisplayComponent implements OnInit {
    user$: Observable<User>;

    constructor(private userService: UserService) { }

    ngOnInit(): void {
        this.user$ = this.userService.getUser();
    }
}
```

Everytime the user email or password changes, the component will be redrawn. This is unnecessary as we only care about the username. So let's introduce a filter.

```typescript
@Component({ 
    template: `{% raw %} 
        <p *ngIf="user$ | async as user">Welcome, {{user?.name}}</p>
    {% endraw %}`, 
    selector:'user-name-display-component'
})
export class UserNameDisplayComponent implements OnInit {
    user$: Observable<User>;

    constructor(private userService: UserService) { }

    ngOnInit(): void {
        this.user$ = this.userService.getUser().pipe(
            distinct((user: User) => user.name)
        );
    }
}
```

Since most operators are pipeable pure functions we can use the `pipe()` method on an observable to pass a list of operators that will be executed in order. In the above example we pass the distinct function, which returns another observable that only emits when the `name` property on the `user` object is different from the previous emission. The end result will be less change detection cycles and a more performant application.

Another common filtering operator is the aptly named `filter` operator, which takes a predicate filter as parameter that in turn takes the value and then returns a boolean, indicating if a particular emission should be further emitted down the chain. 

Let's continue the above example with a filter operator. Say there was a technical user with name 'admin' that our component needs to ignore. Some code might look like this:

```typescript
@Component({ 
    template: `{% raw %}
        <p *ngIf="user$ | async as user">Welcome, {{user?.name}}</p>
    {% endraw %}`, 
    selector:'user-name-display-component'
})
export class UserNameDisplayComponent implements OnInit {
    user$: Observable<User>;

    constructor(private userService: UserService) { }

    ngOnInit(): void {
        this.user$ = this.userService.getUser().pipe(
            filter((user: User) => user.name !== 'admin'),
            distinct((user: User) => user.name)
        );
    }
}
```

As we can see, we just added the `filter` operator in the chain, and thus any emission on the `getUser()` method will first pass the `filter` and then (if the filter is passed) the `distinct` operator. Order matters in operator chains! Only when both filters are passed will the user object be emitted and the subscriber notified of the change.

## <a name="transformation-operators"></a>Transformation operators

Transforming an object to the needs of an application is a highly common use case and the `map` operator is there to do just that. In the example of our `UserNameDisplayComponent`, we actually only care about the username, so there's no need to subscribe to the entire user object.

```typescript
@Component({ 
    template: `{% raw %}
        <p *ngIf="userName$ | async as userName">Welcome, {{userName}}</p>
    {% endraw %}`, 
    selector:'user-name-display-component'
})
export class UserNameDisplayComponent implements OnInit {
    userName$: Observable<string>;

    constructor(private userService: UserService) { }

    ngOnInit(): void {
        this.userName$ = this.userService.getUser().pipe(
            filter((user: User) => user.name !== 'admin'),
            distinct((user: User) => user.name),
            map((user: User) => user.name)
        );
    }
}
```

After passing through our filters (it makes sense to always filter first), we transform the result to an object that suits our needs. Here we only care about one property, so we just return that. As a result, our `UserNameDisplayComponent` only knows about the username.

## <a name="flattening-operators"></a>Flattening Operators

In reality we would be lucky to get all the data we need to display in a single call. We might need to combine data of two or more sources to get the result we need to achieve. That's where flattening operators come in. There are four in total, with `switchMap` being the most widely used as it is considered a safe default. I will go into detail about the other operators later and limit the scope of this post on the general use of `switchMap`.

Let's say the `UserService` only returns the technical user and we needed to retrieve the details from another service. We'll call this the `PersonService` and use the email as the unique identifier for a person. Our interface might look like something like this:

```typescript
interface User {    
    password: string;
    email: string;
}

interface Person {
    email: string;
    name: string;
    age: number;
}
```

The `UserService` remains the same and returns an observable of `User`, and we have a new `PersonService` which contains a `getPersonByEmail(email: string)` method that returns an observable of `Person`. Our component can thus be rewritten as follows:

```typescript
@Component({ 
    template: `{% raw %}
        <p *ngIf="userName$ | async as userName">Welcome, {{userName}}</p>
    {% endraw %}`, 
    selector:'user-name-display-component'
})
export class UserNameDisplayComponent implements OnInit {
    userName$: Observable<string>;

    constructor(
        private personService: PersonService,
        private userService: UserService
    ) { }

    ngOnInit(): void {
        this.userName$ = this.userService.getUser().pipe(
            switchMap(
                (user: User) => this.personService.getPersonByEmail(user.email)
                .pipe(
                    map((person: Person) => person.name)
                )
            )
        );
    }
}
```

What we do here is nest two observables. The `switchMap` operator will trigger a subscription on each emission of the higher order observable (in this case the `getUser()` method). The subscription will invoke the `getPersonByEmail()` method which returns an observable and subsribe to it. Here we do an inner map to retrieve an object that is suitable to our application needs. Note that the `user` variable is still in scope inside the `map` operator and thus can also be used in the map, making it easy to create an object consisting of data present in both results. In this case we only care about the name of the person so we just return it, ignoring the `user` parameter.

The great thing about the `switchMap` operator is that we don't need to manage the inner subscription, the operator takes care of it for us. This is another to reason to never subscribe inside a service but rather at the calling component, we can just take any service method returning an observable and use it in a `switchMap` statement or choose to subscribe inside a component. 

The `switchMap` operator also cancels any previous but not yet completed calls to the new observable, in case multiple emissions from the higher order observable quickly succeed one another. This makes it easy to set up something like a typeahead control, where you send http requests based on input in a text box. 

## <a name="conclusion"></a>Conclusion

For brevity only some of the most common used operators are discussed above. The idea is just to give a general overview of how operators work. Don't forget to check out the [official RxJs docs](https://rxjs-dev.firebaseapp.com/api) to see all available operators.