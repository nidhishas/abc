# Chapter 6: Router State

![](images/4_angular_router_overview/cycle_recognizing.png)

During a navigation, after redirects have been applied, the router creates a `RouterStateSnapshot`.

## What is RouterStateSnapshot?

```javascript
interface RouterStateSnapshot {
  root: ActivatedRouteSnapshot;
}

interface ActivatedRouteSnapshot {
  url: UrlSegment[];
  params: {[name:string]:string};
  data: {[name:string]:any};

  queryParams: {[name:string]:string};
  fragment: string;

  root: ActivatedRouteSnapshot;
  parent: ActivatedRouteSnapshot;
  firstchild: ActivatedRouteSnapshot;
  children: ActivatedRouteSnapshot[];
}
```

As you can see `RouterStateSnapshot` is a tree of activated route snapshots. Every node in this tree knows about the "consumed" URL segments, the extracted parameters, and the resolved data. To make it clearer, let's look at this example:

```javascript
[
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
          {
            path: 'messages',
            component: MessagesCmp
          },
          {
            path: 'messages/:id',
            component: MessageCmp,
            resolve: {
              message: MessageResolver
            }
          }
        ]
      }
    ]
  }
]
```

When we are navigating to `'/inbox/33/messages/44'`, the router will look at the URL and will construct the following `RouterStateSnapshot`:

{width=40%}
![](images/8_routerstate/router_state.png)

After that the router will instantiate `ConversationCmp` with `MessageCmp` in it.

Now imagine we are navigating to a different URL: `'/inbox/33/messages/45'`, which will result in the following snapshot:

{width=40%}
![](images/8_routerstate/router_state2.png)

To avoid unnecessary DOM modifications, the router will reuse the components when the parameters of the corresponding routes change. In this example, the id parameter of the message component has changed from 44 to 45. This means that we cannot just inject an `ActivatedRouteSnapshot` into `MessageCmp` because the snapshot will always have the id parameter set to 44, i.e., it will get stale.

The router state snapshot represents the state of the application at a moment in time, hence the name 'snapshot'. But components can stay active for hours, and the data they show can change. So having only snapshots won't cut it--we need a data structure that allows us to deal with changes.

Introducing RouterState!

```javascript
interface RouterState {
  snapshot: RouterStateSnapshot; //returns current snapshot

  root: ActivatedRoute;
}

interface ActivatedRoute {
  snapshot: ActivatedRouteSnapshot; //returns current snapshot

  url: Observable<UrlSegment[]>;
  params: Observable<{[name:string]:string}>;
  data: Observable<{[name:string]:any}>;

  queryParams: Observable<{[name:string]:string}>;
  fragment: Observable<string>;

  root: ActivatedRout;
  parent: ActivatedRout;
  firstchild: ActivatedRout;
  children: ActivatedRout[];
}
```

`RouterState` and `ActivatedRoute` are similar to their snapshot counterparts except that they expose all the values as observables, which are great for dealing with values changing over time.

Any component instantiated by the router can inject its `ActivatedRoute`.

```javascript
@Component({
  template: `
      Title: {{(message|async).title}}
      ...
  `
})
class MessageCmp {
  message: Observable<Message>;
  constructor(r: ActivatedRoute) {
    this.message = r.data.map(d => d.message);
  }
}
```

If we navigate from `'/inbox/33/messages/44'` to `'/inbox/33/messages/45'`, the data observable will emit a new set of data with the new message object, and the component will display Message 45.

## Accessing Snapshots

The router exposes parameters and data as observables, which is convenient most of the time, but not always. Sometimes what we want is a snapshot of the state that we can examine at once.

```javascript
@Component({...})
class MessageCmp {
  constructor(r: ActivatedRoute) {
    r.url.subscribe(() => {
      r.snapshot; // any time url changes, this callback is fired
    });
  }
}
```

## ActivatedRoute

`ActivatedRoute` provides access to the url, params, data, queryParams, and fragment observables. We will look at each of them in detail, but first let's examine the relationships between them.

URL changes are the source of any changes in a route. And it has to be this way as the user has the ability to modify the location directly.

Any time the URL changes, the router derives a new set of parameters from it: the router takes the positional parameters (e.g., ':id') of the matched URL segments and the matrix parameters of the last matched URL segment and combines those. This operation is pure: the URL has to change for the parameters to change. Or in other words, the same URL will always result in the same set of parameters.

Next, the router invokes the route's data resolvers and combines the result with the provided static data. Since data resolvers are arbitrary functions, the router cannot guarantee that you will get the same object when given the same URL. Even more, often this cannot be the case! The URL contains the id of a resource, which is fixed, and data resolvers fetch the content of that resource, which often varies over time.

Finally, the activated route provides the queryParams and fragment observables. In opposite to other observables, that are scoped to a particular route, query parameters and fragment are shared across multiple routes.

### URL

Given the following:

```javascript
@Component({...})
class ConversationCmp {
  constructor(r: ActivatedRoute) {
    r.url.subscribe((s:UrlSegment[]) => {
      console.log("url", s);
    });
  }
}
```

And navigating first to `'/inbox/33/messages/44'` and then to `'/inbox/33/messages/45'`, we will see:

```
url [{path: 'messages', params: {}}, {path: '44', params: {}}]
url [{path: 'messages', params: {}}, {path: '45', params: {}}]
```

We do not often listen to URL changes as those are too low level. One use case where it can be practical is when a component is activated by a wildcard route. Since in this case the array of URL segments is not fixed, it might be useful to examine it to show different data to the user.


### Params

Given the following:

```javascript
@Component({...})
class MessageCmp {
  constructor(r: ActivatedRoute) {
    r.params.subscribe((p => {
      console.log("params", params);
    });
  }
}
```

And when navigating first to `'/inbox/33/messages;a=1/44;b=1'` and then to `'/inbox/33/messages;a=2/45;b=2'`, we will see

```
params {id: '44', b: '1'}
params {id: '45', b: '2'}
```

First thing to note is that the id parameter is a string (when dealing with URLs we always work with strings). Second, the route gets only the matrix parameters of its last URL segment. That is why the 'a' parameter is not present.

### Data

Let's tweak the configuration from above to see how the data observable works.

```javascript
{
  path: 'messages/:id',
  component: MessageCmp,
  data: {
    allowReplyAll: true
  },
  resolve: {
    message: MessageResolver
  }
}
```

Where `MessageResolver` is defined as follows:

```javascript
class MessageResolver implements Resolve<any> {
  constructor(private repo: ConversationsRepo, private currentUser: User) {}

  resolve(route: ActivatedRouteSnapshot, state: RouteStateSnapshot):
    Promise<Message> {
    return this.repo.fetchMessage(route.params['id'], this.currentUser);
  }
}
```

The data property is used for passing a fixed object to an activated route. It does not change throughout the lifetime of the application. The resolve property is used for dynamic data.

Note that in the configuration above the line `"message: MessageResolver"` does not tell the router to instantiate the resolver. It instructs the router to fetch one using dependency injection. This means that you have to register `MessageResolver` in the list of providers somewhere.

Once the router has fetched the resolver, it will call the 'resolve' method on it. The method can return a promise, an observable, or any other object. If the return value is a promise or an observable, the router will wait for that promise or observable to complete before proceeding with the activation.

The resolver does not have to be a class implementing the `Resolve` interface. It can also be a function:

```javascript
function resolver(route: ActivatedRouteSnapshot, state: RouteStateSnapshot):
  Promise<Message> {
  return repo.fetchMessage(route.params['id'], this.currentUser);
}
```

The router combines the resolved and static data into a single property, which you can access, as follows:

```javascript
@Component({...})
class MessageCmp {
  constructor(r: ActivatedRoute) {
    r.data.subscribe((d => {
      console.log('data', d);
    });
  }
}
```

When navigating first to `'/inbox/33/message/44'` and then to `'/inbox/33/messages/45'`, we will see

```
data {allowReplyAll: true, message: {id: 44, title: 'Rx Rocks', ...}}
data {allowReplyAll: true, message: {id: 45, title: 'Angular Rocks', ...}}
```

## Query Params and Fragment

In opposite to other observables, that are scoped to a particular route, query parameters and fragment are shared across multiple routes.

```javascript
@Component({...})
class MessageCmp {
  debug: Observable<string>;
  fragment: Observable<string>;

  constructor(route: ActivatedRoute) {
    this.debug = route.queryParams.map(p => p.debug);
    this.fragment = route.fragment;
  }
}
```