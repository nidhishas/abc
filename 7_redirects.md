# Chapter 5: Redirects

![](images/4_angular_router_overview/cycle_applying_redirects.png)

Using redirects we can transform the URL before the router creates a router state out of it. This is useful for normalizing URLs and large scale refactorings.

## Local and Absolute Redirects

Redirects can be local and absolute. Local redirects replace a single URL segment with a different one. Absolute redirects replace the whole URL.

If the 'redirectTo' value starts with a '/', then it is an absolute redirect. The next example shows the difference between relative and absolute redirects.

```javascript
[
  {
    path: ':folder/:id',
    component: ConversationCmp,
    children: [
      {
        path: 'contacts/:name',
        redirectTo: '/contacts/:name'
      },
      {
        path: 'legacy/messages/:id',
        redirectTo: 'messages/:id'
      },
      {
        path: 'messages/:id',
        component: MessageCmp
      }
    ]
  },
  {
    path: 'contacts/:name',
    component: ContactCmp
  }
]
```

When navigating to `'/inbox/33/legacy/messages/44'`, the router will apply the second redirect and will change the URL to `'/inbox/33/messages/44'`. In other words, the part of the URL corresponding to the matched segment will be replaced. But navigating to `'/inbox/33/contacts/jim'` will replace the whole URL with `'/contacts/jim'`.

Note that a redirectTo value can contain variable segments captured by the path expression (e.g., ':name', ':id'). All the matrix parameters of the corresponding segments matched by the path expression will be preserved as well.

## One Redirect at a Time

You can set up redirects at different levels of your router configuration. Let's modify the example from above to illustrate this.

```javascript
[
  {
    path: 'legacy/:folder/:id',
    redirectTo: ':folder/:id'
  },
  {
    path: ':folder/:id',
    component: ConversationCmp,
    children: [
      {
        path: 'legacy/messages/:id',
        redirectTo: 'messages/:id'
      },
      {
        path: 'messages/:id',
        component: MessageCmp
      }
    ]
  }
]
```

When navigating to `'/legacy/inbox/33/legacy/messages/44'`, the router will first apply the outer redirect, transforming the URL to `'/inbox/33/legacy/messages/44'`. After that the router will start processing the children of the second route and will apply the inner redirect, resulting in this URL: `'/inbox/33/messages/44'`.

One constraint the router imposes is at any level of the configuration the router applies only one redirect, i.e., redirects cannot be chained.

For instance, say we have this configuration.

```javascript
[
  {
    path: 'legacy/messages/:id',
    redirectTo: 'messages/:id'
  },
  {
    path: 'messages/:id',
    redirectTo: 'new/messages/:id'
  },
  {
    path: 'new/messages/:id',
    component: MessageCmp
  }
]
```

When navigating to `'legacy/messages/:id'`, the router will replace the URL with `'messages/:id'` and will stop there. It won't redirect to `'new/messages/:id'`. A similar constraint is applied to absolute redirects: once an absolute redirect is matched, the redirect phase stops.

## Using Redirects to Normalize URLs

We often use redirects for URL normalization. Say we want both `mail-app.vsavkin.com` and `mail-app.vsavkin.com/inbox` render the same UI. We can use a redirect to achieve that:

```javascript
[
  { path: '', pathMatch: 'full', redirectTo: '/inbox' },
  {
    path: ':folder',
    children: [
      ...
    ]
  }
]
```

We can also use redirects to implement a not-found page.

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
      { path: '**', redirectTo: '/notfound/conversation' }
    ]
  }
  { path: 'notfound/:objectType', component: NotFoundCmp }
]
```

## Using Redirects to Enable Refactoring

Another big use case for using redirects is to enable large scale refactorings. Such refactorings can take months to complete, and, consequently, we will not be able to update all the URLs in the whole application in one go. We will need to do in phases. By using redirects we can keep the old URLs working while migrating to the new implementation.