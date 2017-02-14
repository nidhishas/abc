# Chapter 10: Events

The router provides an observable of navigation events. Any time the user navigates somewhere, or an error is thrown, a new event is emitted. This can be useful for setting up monitoring, troubleshooting issues, implementing error handling, etc.

## Enable Tracing

The very first thing we can do during development to start troubleshoot router-related issues is to enable tracing, which will print out every single event in the console.

```javascript
@NgModule({
  import: [RouterModule.forRoot(routes, {enableTracing: true})]
})
class MailModule {
}
platformBrowserDynamic().bootstrapModule(MailModule);
```

## Listening to Events

To listen to events, inject the router service and subscribe to the events observable.

```javascript
class MailAppCmp {
  constructor(r: Router) {
    r.events.subscribe(e => {
      console.log("event", e);
    });
  }
}
```

For instance, let's say we want to update the title any time the user succesfully navigates. An easy way to do that would be to listen to all `NavigationEnd` events:

```javascript
class MailAppCmp {
  constructor(r: Router, titleService: TitleService) {
    r.events.filter(e => e instanceof NavigationEnd).subscribe(e => {
      titleService.updateTitleForUrl(e.url);
    });
  }
}
```

## Grouping by Navigation ID

The router assigns a unique id to every navigation, which we can use to correlate events.

Let start with defining a few helpers used for identifying the start and the end of a particular navigation.

```javascript
function isStart(e: Event): boolean {
  return e instanceof NavigationStart;
}

function isEnd(e: Event): boolean {
  return e instanceof NavigationEnd ||
         e instanceof NavigationCancel ||
         e instanceof NavigationError;
}
```

 Next, let's define a combinator that will take an observable of all the events related to a navigation and reduce them into an array.

```javascript
function collectAllEventsForNavigation(obs: Observable<Event>):
  Observable<Event[]> {
  let observer: Observer<Event[]>;
  const events = [];
  const sub = obs.subscribe(e => {
    events.push(e);
    if (isEnd(e)) {
      observer.next(events);
      observer.complete();
    }
  });
  return new Observable<Event[]>(o => observer = o);
}
```

Now equipped with these helpers, we can implement the desired functionality.

```javascript
class MailAppCmp {
  constructor(r: Router) {
    r.events.

      // Groups all events by id and returns Observable<Observable<Event>>.
      groupBy(e => e.id).

      // Reduces events and returns Observable<Observable<Event[]>>.
      // The inner observable has only one element.
      map(collectAllEventsForNavigation).

      // Returns Observable<Event[]>.
      mergeAll().

      subscribe((es:Event[]) => {
        console.log("navigation events", es);
      });
  }
}
```

## Showing Spinner

In the last example let's use the events observable to show the spinner during navigation.

```javascript
class MailAppCmp {
  constructor(r: Router, spinner: SpinnerService) {
    r.events.
      // Fitlers only starts and ends.
      filter(e => isStart(e) || isEnd(e)).

      // Returns Observable<boolean>.
      map(e => isStart(e)).

      // Skips duplicates, so two 'true' values are never emitted in a row.
      distinctUntilChanged().

      subscribe(showSpinner => {
        if (showSpinner) {
          spinner.show();
        } else {
          spinner.hide();
        }
      });
  }
}
```