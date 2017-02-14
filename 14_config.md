# Chapter 12: Configuration

In this last chapter we will look at configuring the router.

## Importing RouterModule

We configure the router by importing `RouterModule`, and there are two ways to do it: `RouterModule.forRoot` and `RouterModule.forChild`.

`RouterModule.forRoot` creates a module that contains all the router directives, the given routes, and the router service itself. And `RouterModule.forChild` creates a module that contains all the directives and the given routes, but does not include the service.

The router library provides two ways to configure the module because it deals with a shared mutable resource--location. That is why we cannot have more than one router service active--they would clobber each other. Therefore we can use `forChild` to configure every lazy-loaded child module, and `forRoot` at the root of the application. `forChild` can be called multiple times, whereas `forRoot` can be called only once.

```javascript
@NgModule({
  imports: [RouterModule.forRoot(ROUTES)]
})
class MailModule {}

@NgModule({
  imports: [RouterModule.forChild(ROUTES)]
})
class ContactsModule {}
```

## Configuring Router Service

We can configure the router service by passing the following options to `RouterModule.forRoot`:

* `enableTracing` makes the router log all its internal events to the console.
* `useHash` enables the location strategy that uses the URL fragment instead of the history API.
* `initialNavigation` disables the initial navigation.
* `errorHandler` provides a custom error handler.

Let's look at each of them in detail.

### Enable Tracing

Setting `enableTracing` to true is a great way to learn how the router works.

```javascript
@NgModule({
  imports: [RouterModule.forRoot(ROUTES, {enableTracing: true})]
})
class MailModule {}
```

With this option set, the router will log every internal event to the your console. You'll see something like this:

```
Router Event: NavigationStart
NavigationStart(id: 1, url: '/inbox')

Router Event: RoutesRecognized
RoutesRecognized(id: 1, url: '/inbox', urlAfterRedirects: '/inbox', state:
  Route(url:'', path:'') {
    Route(url:'inbox', path:':folder') {
      Route(url:'', path:'')
    }
  }
)

Router Event: NavigationEnd
NavigationEnd(id: 1, url: '/inbox', urlAfterRedirects: '/inbox')


Router Event: NavigationStart
NavigationStart(id: 2, url: '/inbox/0')

Router Event: RoutesRecognized
RoutesRecognized(id: 2, url: '/inbox/0', urlAfterRedirects: '/inbox/0', state:
  Route(url:'', path:'') {
    Route(url:'inbox', path:':folder') {
      Route(url:'0', path:':id') {
        Route(url:'', path:'')
      }
    }
  }
)

Router Event: NavigationEnd
NavigationEnd(id: 2, url: '/inbox/0', urlAfterRedirects: '/inbox/0')
```

You can right click on any of the events and store them as global variables. This allows you to interact with them: inspect router state snapshots, URLs, etc..

### Use Hash

The router supports two location strategies out of the box: the first one uses the browser history API, and the second one uses the URL fragment or hash. To enable the hash strategy, do the following:

```javascript
@NgModule({
  imports: [RouterModule.forRoot(ROUTES, {useHash: true})]
})
class MailModule {}
```

You can also provide your own custom strategy as follows:

```javascript
@NgModule({
  imports: [RouterModule.forRoot(ROUTES)],
  providers: [{provide: LocationStrategy, useClass: MyCustomLocationStrategy}]
})
class MailModule {}
```

## Disable Initial Navigation

By default, `RouterModule.forRoot` will trigger the initial navigation: the router will read the current URL and will navigate to it. We can disable this behavior to have more control.

```javascript
@NgModule({
  imports: [RouterModule.forRoot(ROUTES, {initialNavigation: false})],
})
class MailModule {
  constructor(router: Router) {
    router.navigateByUrl("/fixedUrl");
  }
}
```

## Custom Error Handler

Every navigation will either succeed, will be canceled, or will error. There are two ways to observe this.

The `router.events` observable will emit:

* `NavigationStart` when navigation stars.
* `NavigationEnd` when navigation succeeds.
* `NavigationCancel` when navigation is canceled.
* `NavigationError` when navigation fails.

All of them contain the `id` property we can use to group the events associated with a particular navigation.

If we call `router.navigate` or `router.navigateByUrl` directly, we will get a promise that:

* will be resolved with `true` if the navigation succeeds.
* will be resolved with `false` if the navigation gets canceled.
* will be rejected if the navigation fails.

Navigation fails when the router cannot match the URL or an exception is thrown during the navigation. Usually this indicates a bug in the application, so failing is the right strategy, but not always. We can provide a custom error handler to recover from certain errors.

```javascript
function treatCertainErrorsAsCancelations(error) {
  if (error isntanceof CancelException) {
    return false; //cancelation
  } else {
    throw error;
  }
}

@NgModule({
  imports: [RouterModule.forRoot(ROUTES, {errorHandler: treatCertainErrorsAsCancelations})]
})
class MailModule {}
```
