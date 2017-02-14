# Chapter 2: Overview

Now that we have learned what routers do in general, it is time to talk about the Angular router.

![](images/4_angular_router_overview/cycle_full.png)

The Angular router takes a URL, then:

1. Applies redirects
2. Recognizes router states
3. Runs guards and resolves data,
4. Activates all the needed components
5. Manages navigation

Most of it happens behind the scenes, and, usually, we do not need to worry about it. But remember, the purpose of this book is to teach you how to configure the router to handle any crazy requirement your application might have. So let's get on it!

### URL Format

Since I will use a lot of URLs in the examples below, let's quickly look at the URL format.

    /inbox/33(popup:compose)

    /inbox/33;open=true/messages/44

As you can see, the router uses parentheses to serialize secondary segments (e.g., popup:compose), the colon syntax to specify the outlet, and the ';parameter=value' syntax (e.g., open=true) to specify route specific parameters.

---

In the examples below we assume that we have given the following configuration to the router, and we are navigating to `'/inbox/33/messages/44'`.

```javascript
[
  { path: '', pathMatch: 'full', redirectTo: '/inbox' },
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
        children: [
          { path: 'messages', component: MessagesCmp },
          { path: 'messages/:id', component: MessageCmp }
        ]
      }
    ]
  },
  {
    path: 'compose',
    component: ComposeCmp,
    outlet: 'popup'
  },
  {
    path: 'message/:id',
    component: PopupMessageCmp,
    outlet: 'popup'
  }
]
```

## Applying Redirects

![](images/4_angular_router_overview/cycle_applying_redirects.png)

The router gets a URL from the user, either when she clicks on a link or updates the location bar directly. The first thing that router does with this URL is it will apply any redirects.

What is a redirect?

I> A redirect is a substitution of a URL segment. Redirects can either be local or absolute. Local redirects replace a single segment with a different one. Absolute redirects replace the whole URL. Redirects are local unless you prefix the url with a slash.

The provided configuration has only one redirect rule: `{ path: '', pathMatch: 'full', redirectTo: '/inbox' }`, i.e., replace `'/'` with `'/inbox'`. This redirect is absolute because the redirectTo value starts with a slash.

Since we are navigating to `'/inbox/33/messages/44'` and not `'/'`, the router will not apply any redirects, and the URL will stay as is.


## Recognizing States

![](images/4_angular_router_overview/cycle_recognizing.png)

Next, the router will derive a router state from the URL. To understand how this phase works, we need to learn a bit about how the router matches the URL.

The router goes through the array of routes, one by one, checking if the URL starts with a route's path. Here it will check that `'/inbox/33/messages/44'` starts with ':folder'. It does, since ‘:folder’ is what is called a ‘variable segment’. It is a parameter where normally you’d expect to find a constant string. Since it is a variable, virtually any string will match it. In our case ‘inbox’ will match it. So the router will set the folder parameter to 'inbox', then it will take the children configuration items, the rest of the URL `'33/messages/44'`, and will carry on matching. As a result, the id parameter will be set to '33', and, finally, the `'messages/:id'` route will be matched with the second id parameter set to '44'.

If the taken path through the configuration does not "consume" the whole URL, the router backtracks to try an alternative path. If it is impossible to match the whole URL, the navigation fails. But if it works, the router state representing the future state of the application will be constructed.

{width=80%}
![](images/4_angular_router_overview/router_state.png)

A router state consists of activated routes. And each activated route can be associated with a component. Also, note that we always have an activated route associated with the root component of the application.


## Running Guards

![](images/4_angular_router_overview/cycle_running_guards.png)

At this stage we have a future router state. Next, the router will check that transitioning to the new state is permitted. It will do this by running guards. We will cover guards in detail in next chapters. For now, it is sufficient to say that a guard is a function that the router runs to make sure that a navigation to a certain URL is permitted.


## Resolving Data

After the router has run the guards, it will resolve the data. To see how it works, let's tweak our configuration from above.

```javascript
[
  {
    path: ':folder',
    children: [
      {
        path: '',
        component: ConversationsCmp,
        resolve: {
          conversations: ConversationsResolver
        }
      }
    ]
  }
]
```

Where `ConversationsResolver` is defined as follows:

```javascript
@Injectable()
class ConversationsResolver implements Resolve<any> {
  constructor(private repo: ConversationsRepo, private currentUser: User) {}

  resolve(route: ActivatedRouteSnapshot, state: RouteStateSnapshot):
      Promise<Conversation[]> {
    return this.repo.fetchAll(route.params['folder'], this.currentUser);
  }
}
```

Finally, we need to register `ConversationsResolver` when bootstrapping our application.

```javascript
@NgModule({
  //...
  providers: [ConversationsResolver],
  bootstrap: [MailAppCmp]
})
class MailModule {
}

platformBrowserDynamic().bootstrapModule(MailModule);
```

Now when navigating to `'/inbox'`, the router will create a router state, with an activated route for the conversations component. That route will have the folder parameter set to 'inbox'. Using this parameter with the current user, we can fetch all the inbox conversations for that user.

We can access the resolved data by injecting the activated route object into the conversations component.

```javascript
@Component({
  template: `
    <conversation *ngFor="let c of conversations | async"></conversation>
  `
})
class ConversationsCmp {
  conversations: Observable<Conversation[]>;
  constructor(route: ActivatedRoute) {
    this.conversations = route.data.pluck('conversations');
  }
}
```

## Activating Components

![](images/4_angular_router_overview/cycle_activation.png)

At this point, we have a router state. The router can now activate this state by instantiating all the needed components, and placing them into appropriate router outlets.

To understand how it works, let's take a look at how we use router outlets in a component template.

The root component of the application has two outlets: primary and popup.

```javascript
@Component({
  template: `
    ...
    <router-outlet></router-outlet>

    ...
    <router-outlet name="popup"></router-outlet>
  `
})
class MailAppCmp {
}
```

Other components, such as `ConversationCmp`, have only one.

```javascript
@Component({
  template: `
    ...
    <router-outlet></router-outlet>
    ...
  `
})
class ConversationCmp {
}
```


Now imagine we are navigating to `'/inbox/33/messages/44(popup:compose)'`.

That's what the router will do. First, it will instantiate `ConversationCmp` and place it into the primary outlet of the root component. Then, it will place a new instance of `ComposeCmp` into the 'popup' outlet. Finally, it will instantiate a new instance of `MessageCmp` and place it in the primary outlet of the just created conversation component.


### Using Parameters

Often components rely on parameters or resolved data. For instance, the conversation component probably need to access the conversation object. We can get the parameters and the data by injecting `ActivatedRoute`.

```javascript
@Component({...})
class ConversationCmp {
  conversation: Observable<Conversation>;
  id: Observable<string>;

  constructor(r: ActivatedRoute) {
    // r.data is an observable
    this.conversation = r.data.map(d => d.conversation);

    // r.params is an observable
    this.id = r.params.map(p => p.id);
  }
}
```

If we navigate from `'/inbox/33/messages/44(popup:compose)'` to

`'/inbox/34/messages/45(popup:compose)'`, the data observable will emit a new 'map' with the new object, and the conversation component will display the information about Conversation 34.

As you can see the router exposes parameters and data as observables, which is convenient most of the time, but not always. Sometimes what we want is a snapshot of the state that we can examine at once.

```javascript
@Component({...})
class ConversationCmp {
  conversation: Conversation;
  constructor(r: ActivatedRoute) {
    const s: ActivatedRouteSnapshot = r.snapshot;
    this.conversation = s.data['conversation']; // Conversation
  }
}
```


## Navigation

![](images/4_angular_router_overview/cycle_navigation.png)

So at this point the router has created a router state and instantiated the components. Next, we need to be able to navigate from this router state to another one. There are two ways to accomplish this: imperatively, by calling `router.navigate`, or declaratively, by using the `RouterLink` directive.

### Imperative Navigation

To navigate imperatively, inject the `Router` service and call navigate.

```javascript
@Component({...})
class MessageCmp {
  public id: string;
  constructor(private route: ActivatedRoute, private router: Router) {
    route.params.subscribe(_ => this.id = _.id);
  }

  openPopup(e) {
    this.router.navigate([{outlets: {popup: ['message', this.id]}}]).then(_ => {
      // navigation is done
    });
  }
}
```

### RouterLink

Another way to navigate around is by using the `RouterLink` directive.

```javascript
@Component({
  template: `
    <a [routerLink]="['/', {outlets: {popup: ['message', this.id]}}]">Edit</a>
  `
})
class MessageCmp {
  public id: string;
  constructor(private route: ActivatedRoute) {
    route.params.subscribe(_ => this.id = _.id);
  }
}
```

This directive will also update the href attribute when applied to an `<a>` link element, so it is SEO friendly and the right-click open-in-new-browser-tab behavior we expect from regular links will work.

## Summary

Let's look at all the operations of the Angular router one more time.

![](images/4_angular_router_overview/cycle_full.png)

When the browser is loading `'/inbox/33/messages/44(popup:compose)'`, the router will do the following. First, it will apply redirects. In this example, none of them will be applied, and the URL will stay as is. Then the router will use this URL to construct a new router state.

{width=80%}
![](images/4_angular_router_overview/router_state.png)

Next, the router will instantiate the conversation and message components.

{width=80%}
![](images/4_angular_router_overview/component_tree.png)

Now, let's say the message component has the following link in its template:

```html
<a [routerLink]="[{outlets: {popup: ['message', this.id]}}]">Edit</a>
```

The router link directive will take the array and will set the href attribute to

 `'/inbox/33/messages/44(popup:message/44)'`.

Now, the user triggers a navigation by clicking on the link. The router will take the constructed URL and start the process all over again: it will find that the conversation and message components are already in place. So no work is needed there. But it will create an instance of `PopupMessageCmp` and place it into the popup outlet. Once this is done, the router will update the location property with the new URL.

That was intense--a lot of information! But we learned quite a few things. We learned about the core operations of the Angular router: applying redirects, state recognition, running guards and resolving data, component activation, and navigation. Finally, we looked at an e2e example showing the router in action.

In the rest of this book we will discuss the same operations one more time in much greater depth.