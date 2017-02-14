# Chapter 11: Testing Router

Everything in Angular is testable, and the router isn't an exception. In this chapter we will look at three ways to test routable components: isolated tests, shallow tests, and integration tests.

## Isolated Tests

It is often useful to test complex components without rendering them. To see how it can be done, let's write a test for the following component:

```javascript
@Component({moduleId: module.id, templateUrl: 'compose.html'})
class ComposeCmp {
  form = new FormGroup({
    title: new FormControl('', Validators.required),
    body: new FormControl('')
  });

  constructor(private route: ActivatedRoute,
              private currentTime: CurrentTime,
              private actions: Actions) {}

  onSubmit() {
    const routerStateRoot = this.route.snapshot.root;
    const conversationRoute = routerStateRoot.firstChild;
    const conversationId = +conversationRoute.params['id'];

    const payload = Object.assign({},
      this.form.value,
      {createdAt: this.currentTime()});

    this.actions.next({
      type: 'reply',
      conversationId: conversationId,
      payload: payload
    });
  }
}
```

compose.html

```html
<form [formGroup]="form" (ngSubmit)="onSubmit()">
  <div>
    Title: <md-input formControlName="title" required></md-input>
    <span *ngIf="form.get('title').touched &&
      form.hasError('required', 'title')">
      required
    </span>
  </div>
  <div>
    Body: <textarea formControlName="body"></textarea>
  </div>
  <button type="submit" [disabled]="form.invalid">Reply</button>
</form>
```

There a few things in this example worth noting:

1. We are using reactive forms in the template of this component. This require us to manually create a form object in the component class, which has a nice consequence: we can test input handling without rendering the template.

2. Instead of modifying any state directly, `ComposeCmp` emits an action, which is processed elsewhere. Thus the isolated test will have to only check that the action has been emitted.

3. `this.route.snapshot.root` returns the root of the router state, and `routerStateRoot.firstChild` gives us the conversation route to read the id parameter from.

Now, let's look at the test.

```javascript
describe('ComposeCmp', () => {
  let actions: BehaviorSubject<any>;
  let time: CurrentTime;

  beforeEach(() => {
    // this subject acts as a "spy"
    actions = new BehaviorSubject(null);

    // dummy implementation of CurrentTime
    time = () => '2016-08-19 9:10AM';
  });

  it('emits a reply action on submit', () => {
    // a fake activated route
    const route = {
      snapshot: {
        root: {
          firstChild: { params: { id: 11 } }
        }
      }
    };
    const c = new ComposeCmp(<any>route, time, actions);

    // performing an action
    c.form.setValue({
      title: 'Categorical Imperative vs Utilitarianism',
      body: 'What is more practical in day-to-day life?'
    });
    c.onSubmit();

    // reading the emitted value from the subject
    // to make sure it matches our expectations
    expect(actions.value.conversationId).toEqual(11);
    expect(actions.value.payload).toEqual({
      title: 'Categorical Imperative vs Utilitarianism',
      body: 'What is more practical in day-to-day life?',
      createdAt: '2016-08-19 9:10AM'
    });
  });
});
```

As you can see, testing routable Angular components in isolation is no different from testing any other JavaScript object.



## Shallow Testing
Testing component classes without rendering their templates works in certain scenarios, but not in all of them. Sometimes we can write a meaningful test only if we render a component's template. We can do that and still keep the test isolated. We just need to render the template without rendering the component's children. This is what is colloquially known as shallow testing.

Let's see this approach in action.

```javascript
@Component(
    {moduleId: module.id, templateUrl: 'conversations.html'})
export class ConversationsCmp {
  folder: Observable<string>;
  conversations: Observable<Conversation[]>;

  constructor(route: ActivatedRoute) {
    this.folder = route.params.pluck<string>('folder');
    this.conversations = route.data.pluck<Conversation[]>('conversations');
  }
}
```

This constructor, although short, may look a bit funky if you are not familiar with RxJS. So let's step through it. First, we pluck `folder` out of the params object, which is equivalent to `route.params.map(p => p['folder'])`. Second, we pluck out `conversations`.

In the template we use the async pipe to bind the two observables. The async pipe always returns the latest value emitted by the observable.

```html
{{folder|async}}

<md-card *ngFor="let c of conversations|async" [routerLink]="[c.id]">
  <h3>
    <a [routerLink]="[c.id]">{{c.title}}</a>
  </h3>
  <p>
    <span class="light">{{c.user.name}} [{{c.user.email}}]</span>
  </p>
</md-card>
```

Now let's look at the test.

```javascript
describe('ConversationsCmp', () => {
  let params: BehaviorSubject<string>;
  let data: BehaviorSubject<any>;

  beforeEach(async(() => {
    params = of({
      folder: 'inbox'
    });

    data = of({
      conversations: [
        {
          id: 1,
          title: 'On the Genealogy of Morals by Nietzsche',
          user: {name: 'Kate', email: 'katez@example.com'}
        },
        {
          id: 2,
          title: 'Ethics by Spinoza',
          user: {name: 'Corin', email: 'corin@example.com'}
        }
      ]
    });

    TestBed.configureTestingModule({
      declarations: [ConversationsCmp],
      providers: [
        { provide: ActivatedRoute, useValue: {params, data} }
      ]
    });
    TestBed.compileComponents();
  }));

  it('updates the list of conversations', () => {
    const f = TestBed.createComponent(ConversationsCmp);
    f.detectChanges();

    expect(f.debugElement.nativeElement).toHaveText('inbox');
    expect(f.debugElement.nativeElement).toHaveText('On the Genealogy of Morals');
    expect(f.debugElement.nativeElement).toHaveText('Ethics');

    params.next({
      folder: 'drafts'
    });

    data.next({
      conversations: [
        { id: 3, title: 'Fear and Trembling by Kierkegaard', user: {name: 'Someone Else', email: 'someonelse@example.com'} }
      ]
    });
    f.detectChanges();

    expect(f.debugElement.nativeElement).toHaveText('drafts');
    expect(f.debugElement.nativeElement).toHaveText('Fear and Trembling');
  });
});
```

First, look at how we configured our testing module. We only declared `ConversationsCmp`, nothing else. This means that all the elements in the template will be treated as simple DOM nodes, and only common directives (e.g., ngIf and ngFor) will be applied. This is exactly what we want. Second, instead of using a real activated route, we are using a fake one, which is just an object with the params and data properties.

## Integration Testing

Finally, we can always write an integration test that will exercise the whole application.

```javascript
describe('integration specs', () => {
  const initialData = {
    conversations: [
      {id: 1, title: 'The Myth of Sisyphus'},
      {id: 2, title: 'The Nicomachean Ethics'}
    ],
    messages: [
      {id: 1, conversationId: 1, text: 'The Path of the Absurd Man'}
    ]
  };

  beforeEach(async(() => {
    TestBed.configureTestingModule({
      // MailModule is an NgModule that contains all application
      // components and the router configuration

      // RouterTestingModule overrides the router and location providers
      // to make them test-friendly.
      imports: [MailModule, RouterTestingModule],

	 providers: [
        { provide: 'initialData', useValue: initialData}
      ]
    });
    TestBed.compileComponents();
  }));

  it('should navigate to a conversation', fakeAsync(() => {
    // get the router from the testing NgModule
    const router = TestBed.get(Router);

    // get the location from the testing NgModule,
    // which is a SpyLocation that comes from RouterTestingModule
    const location = TestBed.get(Location);

    // compile the root component of the app
    const f = TestBed.createComponent(MailAppCmp);

    router.navigateByUrl("/inbox");
    advance(f);

    expect(f.debugElement.nativeElement).toHaveText('The Myth of Sisyphus');
    expect(f.debugElement.nativeElement).toHaveText('The Nicomachean Ethics');

    // find the link
    const c = f.debugElement.query(e => e.nativeElement.textContent === "The Myth of Sisyphus");
    c.nativeElement.click();
    advance(f);

    expect(location.path()).toEqual("/inbox/0");
    expect(f.nativeElement).toHaveText('The Path of the Absurd Man');
  }));
});

function advance(f: ComponentFixture<any>) {
  tick();
  f.detectChanges();
}
```

Even though both the shallow and integration tests render components, these tests are very different in nature. In the shallow test we mocked up every single dependency of a component. In the integration one we did it only with the location service. Shallow tests are isolated, and, as a result, can be used to drive the design of our components. Integration tests are only used to check the correctness.


## Summary

In this chapter we looked at three ways to test Angular components: isolated tests, shallow tests, and integration tests. Each of them have their time and place: isolated tests are a great way to test drive your components and test complex logic. Shallow tests are isolated tests on steroids, and they should be used when writing a meaningful test requires to render a component's template. Finally, integration tests verify that a group of components and services (e.g., the router) work together.
