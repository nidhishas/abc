# Chapter 9: Guards

![](images/4_angular_router_overview/cycle_running_guards.png)

The router uses guards to make sure that navigation is permitted, which can be useful for security, authorization, monitoring purposes.

There are four types of guards: `canLoad`, `canActivate`, `canActivateChild`, and `canDeactivate`. In this chapter we will look at each of them in detail.

## CanLoad

Sometimes, for security reasons, we do not want the user to be able to even see the source code of the lazily loaded bundle if she does not have the right permissions. That's what the `canLoad` guard is for. If a `canLoad` guard returns false, the router will not load the bundle.

Let's take this configuration from the previous chapter and set up a `canLoad` guard to restrict access to contacts:

```javascript
const ROUTES = [
  {
    path: ':folder',
    children: [
      {
        path: '',
        component: ConversationsCmp
      },
      {
        path: ':id',
        component: ConversationCmp,
        children: [...]
      }
    ]
  },
  {
    path: 'contacts',
    canLoad: [CanLoadContacts],
    loadChildren: 'contacts.bundle.js'
  }
];
```

Where `CanLoadContacts` is defined like this:

```javascript
@Injectable()
class CanLoadContacts implements CanLoad {
  constructor(private permissions: Permissions,
              private currentUser: UserToken) {}

  canLoad(route: Route): boolean {
    if (route.path === "contacts") {
      return this.permissions.canLoadContacts(this.currentUser);
    } else {
      return false;
    }
  }
}
```

Note that in the configuration above the line `"canLoad: [CanLoadContacts]"` does not tell the router to instantiate the guard. It instructs the router to fetch `CanLoadContacts` using dependency injection. This means that we have to register `CanLoadContacts` in the list of providers somewhere (e.g., when bootstrapping the application).

```javascript
@NgModule({
  //...
  providers: [CanLoadContacts],
  //...
})
class MailModule {
}
platformBrowserDynamic().bootstrapModule(MailModule);
```

The router will use dependency injection to get an instance of `CanLoadContacts`. After that, the route will call the `canLoad` method, which can return a promise, an observable, or a boolean. If the returned value is a promise or an observable, the router will wait for that promise or observable to complete before proceeding with the navigation. If the returned value is false, the navigation will fail.

We could also use a function with the same signature instead of the class.

```javascript
{
  path: 'contacts',
  canLoad: [canLoad],
  loadChildren: 'contacts.bundle.js'
}

function canLoad(route: Route): boolean {
  // ...
}

@NgModule({
  //...
  providers: [{provide: canLoad, useValue: canLoad}],
  //...
})
class MailModule {
}
platformBrowserDynamic().bootstrapModule(MailModule);
```

Finally, the router will call the `canLoad` guard during any navigation loading contacts, not just the first time, even though the bundle itself will be loaded only once.

## CanActivate

The `canActivate` guard is the default mechanism of adding permissions to the application. To see how we can use it, let's take the example from above and remove lazy loading.

```javascript
const ROUTES = [
  {
    path: ':folder',
    children: [
      {
        path: '',
        component: ConversationsCmp
      },
      {
        path: ':id',
        component: ConversationCmp,
        children: [...]
      }
    ]
  },
  {
    path: 'contacts',
    canActivate: [CanActivateContacts],
    children: [
      { path: '', component: ContactsCmp },
      { path: ':id', component: ContactCmp }
    ]
  }
];
```

Where `CanActivateContacts` is defined like this:

```javascript
@Injectable()
class CanActivateContacts implements CanActivate {
  constructor(private permissions: Permissions,
              private currentUser: UserToken) {}

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot):
    boolean {
    if (route.routerConfig.path === "contacts") {
      return this.permissions.canActivate(this.currentUser);
    } else {
      return false;
    }
  }
}
```

As with canLoad, we need to add `CanActivateContacts` to a list of providers, as follows:

```javascript
@NgModule({
  //...
  providers: [CanActivateContacts],
  //...
})
class MailModule {
}
platformBrowserDynamic().bootstrapModule(MailModule);
```

Note, the signatures of the `canLoad` and `canActivate` guards are different. Since `canLoad` is called *during* the construction of the router state, and `canActivate` is called after, the `canActivate` guard gets more information. It gets its activated route and the whole router state, whereas the `canLoad` guard only gets the route.

As with `canLoad`, the router will call `canActivate` any time an activation happens, which includes any parameters' changes.

## CanActivateChild

The `canActivateChild` guard is similar to `canActivate`, except that it is called when a child of the route is activated, and not the route itself.

Imagine a function that takes a URL and decides if the current user should be able to navigate to that URL, i.e., we would like to check that any navigation is permitted. This is how we can accomplish this by using `canActivateChild`.

```javascript
{
  path: '',
  canActivateChild: [AllowUrl],
  children: [
    {
      path: ':folder',
      children: [
        { path: '', component: ConversationsCmp },
        { path: ':id', component: ConversationCmp, children: [...]}
      ]
    },
    {
      path: 'contacts',
      children: [
        { path: '', component: ContactsCmp },
        { path: ':id', component: ContactCmp }
      ]
    }
  ]
}
```

Where `AllowUrl` is defined like this:

```javascript
@Injectable()
class AllowUrl implements CanActivateChild {
  constructor(private permissions: Permissions,
              private currentUser: UserToken) {}

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot):
    boolean {
    return this.permissions.allowUrl(this.currentUser, state.url);
  }
}
```

Since we placed the guard at the very root of the router configuration, it will be called during any navigation.

## CanDeactivate

The `canDeactivate` guard is different from the rest. Its main purpose is not to check permissions, but to ask for confirmation. To illustrate this let's change the application to ask for confirmation when the user closes the compose dialog with unsaved changes.

```javascript
[
  {
    path: 'compose',
    component: ComposeCmp,
    canDeactivate: [SaveChangesGuard]
    outlet: 'popup'
  }
]
```

Where `SaveChangesGuard` is defined as follows:

```javascript
class SaveChangesGuard implements CanDeactivate<ComposeCmp> {
  constructor(private dialogs: Dialogs) {}

  canDeactivate(component: ComposeCmp, route: ActivatedRouteSnapshot,
                state: RouterStateSnapshot): Promise<boolean> {
    if (component.unsavedChanges) {
      return this.dialogs.unsavedChangesConfirmationDialog();
    } else {
      return Promise.resolve(true);
    }
  }
}
```

`SaveChangesGuard` asks the user to confirm the navigation because all the unsaved changes would be lost. If she confirms, the `unsavedChangesConfirmationDialog` will return false, and the navigation will be canceled.