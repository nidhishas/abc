# Chapter 1: What Do Routers Do?

Before we jump into the specifics of the Angular router, let's talk about what routers do in general.

As you know, an Angular application is a tree of components. Some of these components are reusable UI components (e.g., list, table), and some are application components, which represent screens or some logical parts of the application. The router cares about application components, or, to be more specific, about their arrangements. Let's call such component arrangements router states. So a router state defines what is visible on the screen.


I> A router state is an arrangement of application components that defines what is visible on the screen.

### Router Configuration

The router configuration defines all the potential router states of the application. Let's look at an example.

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

Don't worry about understanding all the details. I will cover them in later chapters. For now, let's depict the configuration as follows:

{width=80%}
![](images/3_routing_is_all_about/router_config.png)

As you can see the router configuration is a tree, with every node representing a route. Some nodes have components associated with them, some do not. We also use color to designate different outlets, where an outlet is a location in the component tree where a component is placed.

### Router State

A router state is a subtree of the configuration tree. For instance, the example below has `ConversationsCmp` activated. We say *activated* instead of *instantiated* as a component can be instantiated only once but activated multiple times (any time its route's parameters change).

{width=80%}
![](images/3_routing_is_all_about/activated_1.png)

Not all subtrees of the configuration tree are valid router states. If a node has multiple children of the same color, i.e., of the same outlet name, only one of them can be active at a time. For instance, `ComposeCmp` and `PopupMessageCmp` cannot be displayed together, but `ConversationsCmp` and `PopupMessageCmp` can. Stands to reason, an outlet is nothing but a location in the DOM where a component is placed. So we cannot place more than one component into the same location at the same time.

### Navigation

The router's primary job is to manage navigation between states, which includes updating the component tree.

I> Navigation is the act of transitioning from one router state to another.

To see how it works, let's look at the following example. Say we perform a navigation from the state above to this one:

{width=80%}
![](images/3_routing_is_all_about/activated_2.png)

Because `ConversationsCmp` is no longer active, the router will remove it. Then, it will instantiate `ConversationCmp` with `MessagesCmp` in it, with `ComposeCmp` displayed as a popup.

### Summary

That's it. The router simply allows us to express all the potential states which our application can be in, and provides a mechanism for navigating from one state to another. The devil, of course, is in the implementation details, but understanding this mental model is crucial for understanding the implementation.

### Isn't it all about the URL?

The URL bar provides a huge advantage for web applications over native ones. It allows us to reference states, bookmark them, and share them with our friends. In a well-behaved web application, any application state transition results in a URL change, and any URL change results in a state transition. In other words, a URL is nothing but a serialized router state. The Angular router takes care of managing the URL to make sure that it is always in-sync with the router state.