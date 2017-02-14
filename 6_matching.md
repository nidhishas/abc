# Chapter 4: URL Matching

At the core of the Angular router lies a powerful URL matching engine, which transforms URLs and converts them into router states. Understanding how this engine works is important for implementing advanced use cases.

Once again, let's use this configuration.

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


First, note that every route is defined by two key parts:

* How it matches the URL.
* What it does once the URL is matched.

*Is is important that the second concern, the action, does not affect the matching.*

And let's say we are navigating to `'/inbox/33/messages/44'`.

This is how matching works:

The router goes through the provided array of routes, one by one, checking if the unconsumed part of the URL starts with a route's path.

Here it checks that `'/inbox/33/messages/44'` starts with `:folder`. It does. So the router sets the folder parameter to 'inbox', then it takes the children of the matched route, the rest of the URL, which is `'33/messages/44'`, and carries on matching.

The router will check that `'33/messages/44'` starts with '', and it does, since we interpret every string to begin with the empty string. Unfortunately, the route does not have any children and we haven't consumed the whole URL. So the router will backtrack to try the next route `path: ':id'`.

This one will work. The id parameter will be set to '33', and finally the `messages/:id` route will be matched, and the second id parameter will be set to '44'.

## Backtracking

Let's illustrate backtracking one more time. If the taken path through the configuration does not “consume” the whole url, the router backtracks to try an alternative path.

Say we have this configuration:

```javascript
[
  {
    path: 'a',
    children: [
      {
        path: 'b',
        component: ComponentB
      }
    ]
  },
  {
    path: ':folder',
    children: [
      {
        path: 'c',
        component: ComponentC
      }
    ]
  }
]
```

When navigating to `'/a/c'`, the router will start with the first route. The `'/a/c'` URL starts with `"path: 'a'"`, so the router will try to match `/c` with `b`. Because it is unable to do that, it will backtrack and will match `'a'` with `":folder"`, and then `'c'` with `"c"`.

## Depth-First

The router doesn't try to find the best match, i.e. it does not have any notion of specificity. It is satisfied with the first one that consumes the whole URL.

```javascript
[
  {
    path: ':folder',
    children: [
      {
        path: 'b',
        component: ComponentB1
      }
    ]
  },
  {
    path: 'a',
    children: [
      {
        path: 'b',
        component: ComponentB2
      }
    ]
  }
]
```

When navigating to `'/a/b'`, the first route will be matched even though the second one appears to be 'specific'.

## Wildcards

We have seen that path expressions can contain two types of segments:

* constant segments (e.g., `path: 'messages'`)
* variable segments (e.g., `path: ':folder'`)

Using just these two we can handle most use cases. Sometimes, however, what we want is the "otherwise" route. The route that will match against any provided URL. That's what wildcard routes are. In the example below we have a wildcard route `{ path: '**', redirectTo: '/notfound' }` that will match any URL that we were not able to match otherwise and will activate `NotFoundCmp`.

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
          { path: 'messages', component: MessagesCmp },
          { path: 'messages/:id', component: MessageCmp }
        ]
      }
    ]
  }
  { path: '**', component: NotFoundCmp }
]
```

The wildcard route will "consume" all the URL segments, so `NotFoundCmp` can access those via the injected `ActivatedRoute`.

## Empty-Path Routes

If you look at our configuration once again, you will see that some routes have the path set to an empty string. What does it mean?

```javascript
[
  {
    path: ':folder',
    children: [
      {
        path: '',
        component: ConversationsCmp
      }
    ]
  }
]
```

By setting 'path' to an empty string, we create a route that instantiates a component but does not “consume” any URL segments. This means that if we navigate to `'/inbox'`, the router will do the following:

First, it will check that `/inbox` starts with `:folder`, which it does. So it will take what is left of the URL, which is '', and the children of the route. Next, it will check that '' starts with '', which it does! So the result of all this is the following router state:

{width=40%}
![](images/6_matching/router_state.png)

Empty path routes can have children, and, in general, behave like normal routes. The only special thing about them is that they inherit matrix parameters of their parents. This means that this URL `/inbox;expand=true` will result in the router state where two activated routes have the expand parameter set to true.

{width=40%}
![](images/6_matching/router_state_merged_params.png)

## Matching Strategies

By default the router checks if the URL starts with the path property of a route, i.e., it checks if the URL is prefixed with the path. This is an implicit default, but we can set this strategy explicitly, as follows:

```javascript
// identical to {path: 'a', component: ComponentA}
{path: 'a', pathMatch: 'prefix', component: ComponentA}
```

The router supports a second matching strategy--full, which checks that the path is "equal" to what is left in the URL. This is mostly important for redirects. To see why, let's look at this example:

```javascript
[
  { path: '', redirectTo: '/inbox' },
  {
    path: ':folder',
    children: [
      ...
    ]
  }
]
```

Because the default matching strategy is prefix, and any URL starts with an empty string, the router will always match the first route. Even if we navigate to `'/inbox'`, the router will apply the first redirect. Our intent, however, is to match the second route when navigating to `'/inbox'`, and redirect to `'/inbox'` when navigating to `'/'`. Now, if we change the matching strategy to 'full', the router will apply the redirect only when navigating to `'/'`.

## Componentless Routes

Most of the routes in the configuration have either the redirectTo or component properties set, but some have neither. For instance, look at `"path: ':folder'"` route in the configuration below.

```javascript
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
}
```

We called such routes 'componentless' routes. Their main purpose is to consume URL segments, provide some data to its children, and do it without instantiating any components.

The parameters captured by a componentless route will be merged into their children's parameters. The data resolved by a componentless route will be merged as well. In this example, both the child routes will have the folder parameter in their parameters maps.

This particular example could have been easily rewritten as follows:

```javascript
[
  {
    path: ':folder',
    component: ConversationsCmp
  },
  {
    path: ':folder/:id',
    component: ConversationCmp,
    children: [
      { path: 'messages', component: MessagesCmp },
      { path: 'messages/:id', component: MessageCmp }
    ]
  }
]
```

We have to duplicate the `:folder` parameter, but overall it works. Sometimes, however, there is no other good option but to use a componentless route.

### Sibling Components Using Same Data

For instance, it is useful to share parameters between sibling components.

In the following example we have two components--`MessageListCmp` and `MessageDetailsCmp`--that we want to put next to each other, and both of them require the message id parameter. `MessageListCmp` uses the id to highlight the selected message, and MessageDetailsCmp uses it to show the information about the message.

One way to model that would be to create a bogus parent component, which both `MessageListCmp` and `MessageDetailsCmp` can get the id parameter from, i.e. we can model this solution with the following configuration:

```javascript
[
  {
    path: 'messages/:id',
    component: MessagesParentCmp,
    children: [
      {
        path: '',
        component: MessageListCmp
      },
      {
        path: '',
        component: MessageDetailsCmp,
        outlet: 'details'
      }
    ]
  }
]
```

With this configuration in place, navigating to `'/messages/11'` will result in this component tree:

{width=80%}
![](images/6_matching/component_tree.png)

This solution has a problem--we need to create the bogus component, which serves no real purpose. That's where componentless routes are a good solution:

```javascript
[
  {
    path: 'messages/:id',
    children: [
      {
        path: '',
        component: MessageListCmp
      },
      {
        path: '',
        component: MessageDetailsCmp,
        outlet: 'details'
      }
    ]
  }
]
```

Now, when navigating to `'/messages/11'`, the router will create the following component tree:

{width=80%}
![](images/6_matching/component_tree2.png)

## Composing Componentless and Empty-path Routes

What is really exciting about all these features is that they compose very nicely. So we can use them together to handle advanced use cases in just a few lines of code.

Let me give you an example. We have learned that we can use empty-path routes to instantiate components without consuming any URL segments, and we can use componentless routes to consume URL segments without instantiating components. What about combining them?

```javascript
[
  {
    path: '',
    canActivate: [CanActivateMessagesAndContacts],
    resolve: {
      token: TokenNeededForBothMessagsAndContacts
    },

    children: [
      {
        path: 'messages',
        component: MesssagesCmp
      },
      {
        path: 'contacts',
        component: ContactsCmp
      }
    ]
  }
]
```

Here we have defined a route that neither consumes any URL segments nor creates any components, but used merely for running guards and fetching data that will be used by both `MesssagesCmp` and `ContactsCmp`. Duplicating these in the children is not an option as both the guard and the data resolver can be expensive asynchronous operations and we want to run them only once.

## Summary

We've learned a lot! First, we talked about how the router does matching. It goes through the provided routes, one by one, checking if the URL starts with a route's path. Then, we learned that the router does not have any notion of specificity. It just traverses the configuration in the depth-first order, and it stops after finding the path matching the whole URL, i.e., the order of routes in the configuration matters. After that, we talked about empty-path routes that do not consume any URL segments, and about componentless routes that do not instantiate any components. We showed how we can use them to handle advanced use cases.