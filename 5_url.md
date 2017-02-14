# Chapter 3: URLs

When using the Angular router, a URL is just a serialized router state. Any state transition results in a URL change, and any URL change results in a state transition. Consequently, any link or navigation creates a URL.

## Simple URL

Let's start with this simple URL `'/inbox/33'`.

This is how the router will encode the information about this URL.

```javascript
const url: UrlSegment[] = [
  {path: 'inbox', params: {}},
  {path: '33', params: {}}
];
```

Where UrlSegment is defined as follows:

```javascript
interface UrlSegment {
  path: string;
  params: {[name:string]:string};
}
```

We can use the ActivatedRoute object to get the URL segments consumed by the route.

```javascript
class MessageCmp {
  constructor(r: ActivatedRoute) {
    r.url.forEach((u: UrlSegment[]) => {
      //...
    });
  }
}
```

## Params

Let's soup it up a little by adding matrix or route-specific parameters, so the result URL looks like this: `'/inbox;a=v1/33;b1=v1;b2=v2'`.

```javascript
[
  {path: 'inbox', params: {a: 'v1'}},
  {path: '33', params: {b1: 'v1', b2: 'v2'}}
]
```

Matrix parameters are scoped to a particular URL segment. Because of this, there is no risk of name collisions.

## Query Params

Sometimes, however, you want to share some parameters across many activated routes, and that's what query params are for. For instance, given this URL `'/inbox/33?token=23756'`, we can access 'token' in any component:

```javascript
class ConversationCmp {
  constructor(r: ActivateRoute) {
    r.queryParams.forEach((p) => {
      const token = p['token']
    });
  }
}
```

Since query parameters are not scoped, they should not be used to store route-specific information.

The fragment (e.g., `'/inbox/33#fragment'`) is similar to query params.

```javascript
class ConversationCmp {
  constructor(r: ActivatedRoute) {
    r.fragment.forEach((f:string) => {

    });
  }
}
```

## Secondary Segments

Since a router state is a tree, and the URL is nothing but a serialized state, the URL is a serialized tree. In all the examples so far every segment had only one child. For instance in `'/inbox/33'` the '33' segment is a child of 'inbox', and 'inbox' is a child of the '/' root segment. We called such children 'primary'. Now look at this example:

    /inbox/33(popup:message/44)

Here the root has two children 'inbox' and 'message'.

{width=60%}
![](images/5_url/url_1.png)

The router encodes multiple secondary children using a '//'.

    /inbox/33(popup:message/44//help:overview)

{width=80%}
![](images/5_url/url_2.png)

If some other segment, not the root, has multiple children, the router will encode it as follows:

    /inbox/33/(messages/44//side:help)

{width=60%}
![](images/5_url/url_3.png)