# Chapter 8: Lazy Loading

Angular is built with the focus on mobile. That's why we put a lot of effort into making compiled and bundled Angular 2 applications small. One of the techniques we use extensively is dead code elimination, which helped drop the size of a hello world application to only 20K. This is half the size of an analogous Angular 1 application--an impressive result!

At some point, however, our application will be big enough, that even with this technique, the application file will be too large to be loaded at once. That's where lazy loading comes into play.

Lazy loading speeds up our application load time by splitting it into multiple bundles, and loading them on demand. We designed the router to make lazy loading simple and easy.

## Example

We are going to continue using the mail app example, but this time we will add a new section, contacts, to our application. At launch, our application displays messages. Click the contacts button and it shows the contacts.

Let's start by sketching out our application.

    main.ts:

```javascript
import {Component, NgModule} from '@angular/core';
import {RouterModule} from '@angular/router';
import {platformBrowserDynamic} from '@angular/platform-browser-dynamic';

@Component({...}) class MailAppCmp {}
@Component({...}) class ConversationsCmp {}
@Component({...}) class ConversationCmp {}

@Component({...}) class ContactsCmp {}
@Component({...}) class ContactCmp {}

const ROUTES = [
  {
    path: 'contacts',
    children: [
      { path: '', component: ContactsCmp },
      { path: ':id', component: ContactCmp }
    ]
  },
  {
    path: ':folder',
    children: [
      { path: '', component: ConversationsCmp },
      { path: ':id', component: ConversationCmp, children: [...]}
    ]
  }
];


@NgModule({
  //...
  bootstrap: [MailAppCmp],
  imports: [RouterModule.forRoot(ROUTES)]
})
class MailModule {}

platformBrowserDynamic().bootstrapModule(MailModule);
```

The button showing the contacts UI can look like this:

```html
<button [routerLink]="/contacts">Contacts</button>
```

In addition, we can also support linking to individual contacts, as follows:

```html
<a [routerLink]="['/contacts', id]">Show Contact</a>
```

In the code sample above all the routes and components are defined together, in the same file. This is done for the simplicity sake. How the components are arranged does not really matter, as long as after the compilation we will have a single bundle file `'main.bundle.js'`, which will contain the whole application.

{width=40%}
![](images/10_lazy_loading/lazy_loading_1.png)

### Just One Problem

There is one problem with this setup: even though `ContactsCmp` and `ContactCmp` are not displayed on load, they are still bundled up with the main part of the application. As a result, the initial bundle is larger than it could have been.

Two extra components may not seem like a big deal, but in a real application the contacts module can include dozens or even hundreds of components, together with all the services and helper functions they need.

A better setup would be to extract the contacts-related code into a separate module and load it on-demand. Let's see how we can do that.

## Lazy Loading

We start with extracting all the contacts-related components and routes into a separate file.

    contacts.ts:

```javascript
import {NgModule, Component} from '@angular/core';
import {RouterModule} from '@angular/router';

@Component({...}) class ContactsComponent {}
@Component({...}) class ContactComponent {}

const ROUTES = [
  { path: '', component: ContactsComponent },
  { path: ':id', component: ContactComponent }
];

@NgModule({
  imports: [RouterModule.forChild(ROUTES)]
})
class ContactsModule {}
```

In Angular an ng module is part of an application that can be bundled and loaded independently. So we have defined one in the code above.

### Referring to Lazily-Loaded Module

Now, after extracting the contacts module, we need to update the main module to refer to the newly extracted one.

```javascript
const ROUTES = [
  {
    path: 'contacts',
    loadChildren: 'contacts.bundle.js',
  },
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
  }
];

@NgModule({
  //...
  bootstrap: [MailAppCmp],
  imports: [RouterModule.forRoot(ROUTES)]
})
class MailModule {}

platformBrowserDynamic().bootstrapModule(MailModule);
```

The `loadChildren` properly tells the router to fetch the `'contacts.bundle.js'` when and only when the user navigates to 'contacts', then merge the two router configurations, and, finally, activate the needed components.

By doing so we split the single bundle into two.

{width=80%}
![](images/10_lazy_loading/lazy_loading_2.png)

The bootstrap loads just the main bundle. The router won't load the contacts bundle until it is needed.

```html
<button [routerLink]="/contacts">Contacts</button>

<a [routerLink]="['/contacts', id]">Show Contact</a>
```

Note that apart from the router configuration we don't have to change anything in the application after splitting it into multiple bundles: existing links and navigations are unchanged.

## Deep Linking

But it gets better! The router also supports deep linking into lazily-loaded modules.

To see what I mean imagine that the contacts module lazy loads another one.

    contacts.ts:

```javascript
import {Component, NgModule} from '@angular/core';
import {RouterModule} from '@angular/router';

@Component({...}) class AllContactsComponent {}
@Component({...}) class ContactComponent {}

const ROUTES = [
  { path: '', component: ContactsComponent },
  { path: ':id', component: ContactComponent, loadChildren: 'details.bundle.js' }
];

@NgModule({
  imports: [RouterModule.forChild(ROUTES)]
})
class ContactsModule {}

details.ts:

@Component({...}) class BriefComponent {}
@Component({...}) class DetailComponent {}

const ROUTES = [
  { path: '', component: BriefDetailComponent },
  { path: 'detail', component: DetailComponent },
];

@NgModule({
  imports: [RouterModule.forChild(ROUTES)]
})
class DetailModule {}
```

![](images/10_lazy_loading/lazy_loading_3.png)

Imagine we have the following link in the main section or our application.

```html
<a [routerLink]="['/contacts', id, 'detail', {full: true}]">
  Show Contact Detail
</a>
```

When clicking on the link, the router will first fetch the contacts module, then the details module. After that it will merge all the configurations and instantiate the needed components. Once again, from the link's perspective it makes no difference how we bundle our application. It just works.

## Sync Link Generation

The `RouterLink` directive does more than handle clicks. It also sets the `<a>` tag's href attribute, so the user can right-click and "Open link in a new tab".

For instance, the directive above will set the anchor's href attribute to `'/contacts/13/detail;full=true'`. And it will do it synchronously, without loading the configurations from the contacts or details bundles. Only when the user actually clicks on the link, the router will load all the needed configurations to perform the navigation.

## Navigation is URL-Based

Deep linking into lazily-loaded modules and synchronous link generation are possible only because the router's navigation is URL-based. Because the router does not have the notion of route names, it does not have to use any configuration to generate links. What we pass to routerLink (e.g., `['/contacts', id, 'detail', {full: true}]`) is just an array of URL segments. In other words, link generation is purely mechanical and application independent.

This is an important design decision we have made early on because we knew that lazy loading is a key use case for using the router.


## Customizing Module Loader

The built-in application module loader uses SystemJS. But we can provide our own implementation of the loader as follows:

```javascript
@NgModule({
  //...
  bootstrap: [MailAppCmp],
  imports: [RouterModule.forRoot(ROUTES)],
  providers: [{provide: NgModuleFactoryLoader, useClass: MyCustomLoader}]
})
class MailModule {}

platformBrowserDynamic().bootstrapModule(MailModule);
```

You can look at [`SystemJsNgModuleLoader`](https://github.com/angular/angular/blob/master/modules/@angular/core/src/linker/system_js_ng_module_factory_loader.ts) to see an example of a module loader.

Finally, you don't have to use the loader at all. Instead, you can provide a callback the route will use to fetch the module.

```javascript
{
  path: 'contacts',
  loadChildren: () => System.import('somePath'),
}
```

## Preloading Modules

Lazy loading speeds up our application load time by splitting it into multiple bundles, and loading them on demand. We designed the router to make lazy loading transparent, so you can opt in and opt out of lazy loading with ease.

The issue with lazy loading, of course, is that when the user navigates to the lazy-loadable section of the application, the router will have to fetch the required modules from the server, which can take time.

To fix this problem we have added support for preloading. Now the router can preload lazy-loadable modules in the background while the user is interacting with our application.

This is how it works.

First, we load the initial bundle, which contains only the components we have to have to bootstrap our application. So it is as fast as it can be.

![](images/10_lazy_loading/preloading_1.png)

Then, we bootstrap the application using this small bundle.

![](images/10_lazy_loading/preloading_2.png)

At this point the application is running, so the user can start interacting with it. While she is doing it, we, in the background, preload other modules.

![](images/10_lazy_loading/preloading_3.png)

Finally, when she clicks on a link going to a lazy-loadable module, the navigation is instant.

![](images/10_lazy_loading/preloading_4.png)

We got the best of both worlds: the initial load time is as small as it can ben, and subsequent navigations are instant.

### Enabling Preloading

To enable preloading we need to pass a preloading strategy into forRoot.

```js
@NgModule({
  bootstrap: [MailAppCmp],
  imports: [RouterModule.forRoot(ROUTES,
    {preloadingStrategy: PreloadAllModules})]
})
class MailModule {}
```

The latest version of the router ships with two strategies: preload nothing and preload all modules, but you can provide you own. And it is actually a lot simpler that it may seem.

### Custom Preloading Strategy

Say we don't want to preload all the modules. Rather, we would like to say explicitly, in the router configuration, what should be preloaded.

```js
[
  {
    path: 'moduleA',
    loadChildren: './moduleA.module',
    data: {preload: true}
  },
  {
    path: 'moduleB',
    loadChildren: './moduleB.module'
  }
]
```

We start with creating a custom preloading strategy.

```js
export class PreloadSelectedModulesList implements PreloadingStrategy {
  preload(route: Route, load: Function): Observable<any> {
    return route.data && route.data.preload ? load() : of(null);
  }
}
```

The preload method takes two parameters: a route and the function that actually does the preloading. In it we check if the preload property is set to true. And if it is, we call the load function.

Finally, we need to enable the strategy by listing it as a provider and passing it to RouterModule.forRoot.

```js
@NgModule({
  bootstrap: [MailAppCmp],
  providers: [CustomPreloadingStrategy],
  imports: [RouterModule.forRoot(ROUTES,
    {preloadingStrategy: CustomPreloadingStrategy})]
})
class MailModule {}
```