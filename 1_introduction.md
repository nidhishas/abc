# Introduction

### About the Author

Victor Savkin is an Angular core team member. He is one of the main contributors to Angular and  is the main contributor to the Angular router. Apart from building Angular, Victor also likes to toy with eclectic programming technologies and obsesses over fonts and keyboards.

### Nrwl.io - Angular consulting for enterprise customers, from core team members

Victor is co-founder of Nrwl, a company providing Angular consulting for enterprise customers, from core team members. Visit [nrwl.io](http://nrwl.io) for more information.

![](images/1_introduction/nrwl_logo.png)

### What is this book about?


Managing state transitions is one of the hardest parts of building applications. This is especially true on the web, where you also need to ensure that the state is reflected in the URL. In addition, we often want to split applications into multiple bundles and load them on demand. Doing this transparently is not trivial.

The Angular router solves these problems. Using the router, you can declaratively specify application states, manage state transitions while taking care of the URL, and load bundles on demand.

In this book I will talk about the router's mental model, its API, and the design principles behind it. To make one thing clear: this book is not about Angular. There is a lot of information about the framework available online. So if you want to get familiar with the framework first, I would recommend the following resources:

1. [angular.io](http://angular.io). It's the best place to get started.
2. [egghead.io](https://egghead.io/courses/angular-2-fundamentals). The Angular Fundamentals on egghead.io is an excellent way to get started for those who learn better by watching.
3. [vsavkin.com](http://vsavkin.com). My blog contains many articles where I write about Angular in depth.

Therefore in this book I assume you are familiar with Angular, and so I won't talk about dependency injection, components, or bindings. I will only talk about the router.

### Why would you read this book?

Why would you read this book if you can find the information about the router online? Well, there are a couple of reasons.

First, this book will be kept up to date with the future releases of Angular and the router. All the information will remain accurate and the examples will remain working. This book will never be stale.

Second, the book goes far beyond a how-to-get-started guide. It is a complete description of the Angular router. The mental model, design constraints, and the subtleties of the API-everything is covered. Understanding these will give you deep insights into why the router works the way it does and will make you an Angular router expert.

Let's get started!