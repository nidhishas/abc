# Chapter 7: Links and Navigation

![](images/4_angular_router_overview/cycle_navigation.png)

The primary job of the router is to manage navigation between different router states. There are two ways to accomplish this: imperatively, by calling `router.navigate`, or declaratively, by using the `RouterLink` directive.

As before, let's assume the following configuration:

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

## Imperative Navigation

To navigate imperatively, inject the Router service and call `navigate` or `navigateByUrl` on it. Why two methods and not one?

Using `router.navigateByUrl` is similar to changing the location bar directly--we are providing the "whole" new URL. Whereas `router.navigate` creates a new URL by applying a series of passed-in commands, a patch, to the current URL.

To see the difference clearly, imagine that the current URL is `'/inbox/11/messages/22(popup:compose)'`.

With this URL, calling `router.navigateByUrl('/inbox/33/messages/44')` will result in
`'/inbox/33/messages/44'`, and calling `router.navigate('/inbox/33/messages/44')` will result in
 `'/inbox/33/messages/44(popup:compose)'`.

### Router.navigate

Let's see what we can do with `router.navigate`.


####Passing an array or a string

Passing a string is sugar for passing an array with a single element.

```javascript
router.navigate('/inbox/33/messages/44')
```

is sugar for

```javascript
router.navigate(['/inbox/33/messages/44'])
```

which itself is sugar for

```javascript
router.navigate(['/inbox', 33, 'messages', 44])
```

####Passing matrix params

```javascript
router.navigate([
  '/inbox', 33, {details: true}, 'messages', 44, {mode: 'preview'}
])
```

navigates to

```javascript
'/inbox/33;details=true/messages/44;mode=preview(popup:compose)'
```

#### Updating secondary segments

```javascript
router.navigate([{outlets: {popup: 'message/22'])
```
navigates to

```javascript
'/inbox/11/messages/22(popup:message/22)'
```

We can also update multiple segments at once, as follows:

```javascript
router.navigate([
  {outlets: {primary: 'inbox/33/messages/44', popup: 'message/44'}}
])
```

navigates to

```javascript
'/inbox/33/messages/44(popup:message/44)'
```

And, of course, this works for any segment, not just the root one.

```javascript
router.navigate([
  '/inbox/33', {outlets: {primary: 'messages/44', help: 'message/123'}}
])
```

navigates to

```javascript
'/inbox/33/(messages/44//help:messages/123)(popup:message/44)'
```

We can also remove segments by setting them to 'null'.

```javascript
router.navigate([{outlets: {popup: null])
```

navigates to

```javascript
'/inbox/11/messages/22'
```

####Relative Navigation

By default, the navigate method applies the provided commands starting from the root URL segment, i.e., the navigation is absolute. We can make it relative by providing the starting route, like this:

```javascript
@Component({...})
class MessageCmp {
  constructor(private route: ActivatedRoute, private router: Router) {}

  goToConversation() {
    this.router.navigate('../../', {relativeTo: this.route});
  }
}
```

Let's look at a few examples, given we are providing the `"path: 'message/:id'"` route.

```javascript
router.navigate('details', {relativeTo: this.route})
```

navigates to

```javascript
'/inbox/33/messages/44/details(popup:compose)'
```

The '../' allows us to skip one URL segment.

```javascript
router.navigate('../55', {relativeTo: this.route})
```

navigates to

```javascript
'/inbox/33/messages/55(popup:compose)'
```

```javascript
router.navigate('../../', {relativeTo: this.route})
```

navigates to

```javascript
'/inbox/33(popup:compose)'
```


####Forcing Absolute Navigation

If the first command starts with a slash, the navigation is absolute regardless if we provide a route or not.


####Navigation is URL-Based

Using '../' does not skip a route, but skips a URL segment. More generally, the router configuration has no affect on URL-generation. The router only looks at the current URL and the provided commands to generate a new URL.


####Passing Query Params and Fragment

By default the router resets query params during navigation. If this is the current URL:

```javascript
'/inbox/11/messages/22?debug=true#section2'
```

Then calling

```javascript
router.navigate('/inbox/33/message/44')
```

will navigate to

```javascript
'/inbox/33/messages/44'
```

If we want to preserve the current query params and fragment, we can do the following:

```javascript
router.navigate('/inbox/33/message/44',
  {preserveQueryParams: true, preserveFragment: true})
```

Or we always can provide new params and fragment, like this:

```javascript
router.navigate('/inbox/33/message/44',
  {queryParams: {debug: false}, fragment: 'section3'})
```


### RouterLink

Another way to navigate around is by using the `RouterLink` directive.

```javascript
@Component({
  template: `
    <a [routerLink]="/inbox/33/messages/44">Open Message 44</a>
  `
})
class SomeComponent {}
```

Behind the scenes `RouterLink` just calls `router.navigate` with the provided commands. If we do not start the first command with a slash, the navigation is relative to the route of the component.

Everything that we can pass to `router.navigate`, we can pass to `routerLink`. For instance:

```html
<a [routerLink]="[{outlets: {primary: 'inbox/33/messages/44', popup: 'message/44'}}]">
  Open Message 44
</a>
```

As with `router.navigate`, we can set or preserve query params and fragment.

```html
<a [routerLink]="/inbox/33/messages/44" preserveQueryParams preserveFragment>
  Open Message 44
</a>

<a [routerLink]="/inbox/33/messages/44"
  [queryParams]="{debug:false}" [fragment]="section3">
  Open Message 44
</a>
```

This directive will also update the href attribute when applied to an `<a>` link element, so it is SEO friendly and the right-click open-in-new-browser-tab behavior we expect from regular links will work.

### Active Links

By using the `RouterLinkActive` directive, we can add a CSS class to an element when the link's route becomes active.

```html
<a [routerLink]="/inbox" routerLinkActive="active-link">Inbox</a>
```

When the URL is either `'/inbox'` or `'/inbox/33'`, the active-link class will be added to the `a` tag. If the url changes, the class will be removed.

We can set more than one class, as follows:

```html
<a [routerLink]="/inbox" routerLinkActive="class1 class2">Inbox</a>

<a [routerLink]="/inbox" routerLinkActive="['class1', 'class2']">Inbox</a>
```


####Exact Matching

We can make the matching exact by passing `"{exact: true}"`. This will add the classes only when the URL matches the link exactly. For instance, in the following example the 'active-link' class will be added only when the URL is `'/inbox'`, not `'/inbox/33'`.

```html
<a [routerLink]="/inbox" routerLinkActive="active-link"
  [routerLinkActiveOptions]="{exact: true}">
  Inbox
</a>
```

####Adding Classes to Ancestors

Finally, we can apply this directive to an ancestor of a `RouterLink`.

```html
<div routerLinkActive="active-link" [routerLinkActiveOptions]="{exact: true}">
  <a [routerLink]="/inbox">Inbox</a>
  <a [routerLink]="/drafts">Drafts</a>
</div>
```

This will set the 'active-link' class on the div tag if the URL is either '/inbox' or '/drafts'.

## Summary

That's a lot of information! So let's recap.

First, we established that the primary job of the router is to manage navigation. And there are two ways to do it: imperatively, by calling `router.navigate`, or declaratively, by using the RouterLink directive. We learned about the difference between `router.navigateByUrl` and `router.navigate`: one takes the whole URL, and the other one takes a patch it applies to the current URL. Then we talked about absolute and relative navigation. Finally, we saw how to use `[routerLink]` and `[routerLinkActive]` to set up navigation in the template.