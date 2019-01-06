---
title: "Using NgRx Store Auth Token with Angular HttpInterceptor"
date: 2018-06-04
tags: [angular,ngrx,store,auth,jwt,http,httpclient,interceptor,httpinterceptor]
---

When using NgRx store, it's likely that you will save authentication tokens, such as a JWT, in the store. It's also likely that you want to send this token for many different requests that require authentication. Angular 4.3 introduced `HttpInterceptor`. This allows for registering an interceptor that will be called upon each request where the interceptor is injected.

To mix the two concepts together requires a bit of RxJS `flatMap`, since the intercept method on the interceptor interface takes the request object and handler and returns an observable. This is added to the chain of observables when making HttpClient requests.

Firstly, create the new interceptor.

```typescript
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {

  }
}
```

Next, inject the NgRx Store for access to the token selector through the constructor parameters via dependency injection. I'll assume you have an existing token selector.

```typescript
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  constructor(private store: Store<fromAuth.State>) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {

  }
}
```

The observable that we return from the intercept method begins with the store selector. Since this observable will form part of the chain when creating a new HttpClient request and subscribing, we don't want to send an update everytime the token changes in the future, else the services using HttpClient will appear to get a new value from their request if not unsubscribed. Thus we use the `first()` operator here to only take the first value, then complete.

```typescript
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  constructor(private store: Store<fromAuth.State>) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return this.store.select(fromAuth.getToken).pipe(
      first(),
      ...
    );
  }
}
```

Finally, the logic itself via the `flatMap` operator, which will take our token value from the selector observable and allow us to return a new observable, which is provided by the HttpHandler, using a clone of the request as a parameter. This clone has the `Authorization` header set with our token as the value. Here I'm using a JWT and therefore prefix with `Bearer`.

```typescript
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  constructor(private store: Store<fromAuth.State>) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return this.store.select(fromAuth.getToken).pipe(
      first(),
      flatMap(token => {
        const authReq = !!token ? req.clone({
          setHeaders: { Authorization: 'Bearer ' + token },
        }) : req;
        return next.handle(authReq);
      }),
    );
  }
}
```

With the interceptor complete, the final thing to do is provide the interceptor to the `HTTP_INTERCEPTORS` InjectionToken where we want to inject it, such as the NgModule or service that we want to add our token to requests for.

```typescript
providers: [
  {provide: HTTP_INTERCEPTORS, useClass: TokenInterceptor, multi: true},
],
```
