---
layout:     post
title:      "Custom Validators in Angular 2"
imageUrl:  "/images/custom-validators-bg.jpeg"

date: 2016-03-14
update_date: 2016-05-12

relatedLinks:
  -
    title: "Multi Providers in Angular 2"
    url: "http://blog.thoughtram.io/angular2/2015/11/23/multi-providers-in-angular-2.html"
  -
    title: "Forward References in Angular 2"
    url: "http://blog.thoughtram.io/angular/2015/09/03/forward-references-in-angular-2.html"
  -
    title: "Dependency Injection in Angular 2"
    url: "http://blog.thoughtram.io/angular/2015/05/18/dependency-injection-in-angular-2.html"

summary: "Forms are part of almost every web application out there. Angular strives for making working with forms a breeze. Often, we need to add custom validation capabilities to our application's form. In this article we're going to explore how custom validators can be implemented in Angular 2."

categories:
  - angular
tags:
  - angular2

topic: forms

author: pascal_precht
---

Forms are part of almost every web application out there. Angular strives for making working with forms a breeze. While there are a couple of built-in validators provided by the framework, we often need to add some custom validation capabilities to our application's form, in order to fulfill our needs.

We can easily extend the browser vocabulary with additional custom validators and in this article we are going to explore how to do that.

## Built-in Validators

Angular comes with a subset of built-in validators out of the box. We can apply them either declaratively as directives on elements in our DOM, in case we're building a <strong>template-driven</strong> form, or imperatively using the `Control` and `ControlGroup` or `FormBuilder` APIs, in case we're building a <strong>model-driven</strong> form. If you don't know what it's all about with template-driven and model-driven forms, don't worry, we'll have an article to each of those soon.

The supported built-in validators, at the time of writing this article, are:

- **required** - Requires a form control to have a non-empty value
- **minlength** - Requires a form control to have a value of a minimum length
- **maxlength** - Requires a form control to have a value of a maximum length
- **pattern** - Requires a form control's value to match a given regex

As mentioned earlier, validators can be applied by simply using their corresponding directives. The following form shows how the built-in validators are applied to dedicated form controls:


{% highlight html %}
{% raw %}
<form novalidate>
  <input type="text" ngControl="name" required>
  <input type="text" ngControl="street" minlength="3">
  <input type="text" ngControl="city" maxlength="10">
  <input type="text" ngControl="zip" pattern="[A-Za-z]{5}">
</form>
{% endraw %}
{% endhighlight %}

Or, if our form is model-driven we'd either build it using `Control` and `ControlGroup` APIs:

{% highlight js %}
{% raw %}
@Component()
class Cmp {

  form: ControlGroup;

  ngOnInit() {
    this.form = new ControlGroup({
      name: new Control('', Validators.required)),
      street: new Control('', Validators.minLength(3)),
      city: new Control('', Validators.maxLength(10)),
      zip: new Control('', Validators.pattern('[A-Za-z]{5}'))
    });
  }
}
{% endraw %}
{% endhighlight %}

Or use the less verbose `FormBuilder` API that does the same work for us:

{% highlight js %}
{% raw %}
@Component()
class Cmp {

  constructor(private fb: FormBuilder) {}

  ngOnInit() {
    this.form = this.fb.group({
      name: ['', Validators.required],
      street: ['', Validators.minLength(3)],
      city: ['', Validators.maxLength(10)],
      zip: ['', Validators.pattern('[A-Za-z]{5}')]
    });
  }
}
{% endraw %}
{% endhighlight %}

We would still need to associate a form model with a form in the DOM using the `[ngFormModel]` directive likes this:


{% highlight html %}
{% raw %}
<form novalidate [ngFormModel]="form">
  ...
</form>
{% endraw %}
{% endhighlight %}

Observing these two to three different methods of creating a form, we might wonder how it is done that we can use the validator methods imperatively in our component code, and apply them as directives to input controls declaratively in our HTML code.

It turns out there's really not such a big magic involved, so let's build our own custom email validator.

## Building a custom validator

In it's simplest form, a validator is really just a function that takes a `Control` and returns either `null` when it's valid, or and error object if it's not. A TypeScript interface for such a validator looks something like this:

{% highlight js %}
{% raw %}
interface Validator<T extends Control> {
   (c:T): {[error: string]:any};
}
{% endraw %}
{% endhighlight %}

Let's implement a validator function `validateEmail` which implements that interface. All we need to do is to define a function that takes a `Control`, checks if it's value matches the regex of an email address, and if not, returns an error object,  or `null` in case the value is valid.

Here's what such an implementation could look like:


{% highlight js %}
{% raw %}
import {Control} from '@angular/common';

function validateEmail(c: Control) {
  let EMAIL_REGEXP = ...

  return EMAIL_REGEXP.test(c.value) ? null : {
    validateEmail: {
      valid: false
    }
  };
}
{% endraw %}
{% endhighlight %}

Pretty straight forward right? We import `Control` from `@angular/common` to have the type information the function's signature and simply test a regular expression with the `Control`'s value. That's it. That's a validator.

But how do we apply them to other form controls? Well, we've seen how `Validators.required` and the other validators are added to the `new Control()` calls. `Control()` takes an initial value, a synchronous validator and an asynchronous validator. Which means, we do exactly the same with our custom validators.

{% highlight js %}
{% raw %}
ngOnInit() {
  this.form = new ControlGroup({
    ...
    email: new Control('', validateEmail)
  });
}
{% endraw %}
{% endhighlight %}

Don't forget to import `validateEmail` accordinlgy, if necessary. Okay cool, now we know how to add our custom validator to a form control.

However, what if we want to combine multiple validators on a single control? Let's say our email field is `required` **and** needs to match the shape of an email address. Since `Control`s only take single synchronous and a single asynchronous validator, we need to use `Validators.compose([validator1, validator2])` to compose mutliple validators into one.

Here's what it looks like if we'd combine the `required` validator with our custom one:

{% highlight js %}
{% raw %}
ngOnInit() {
  this.form = new ControlGroup({
    ...
    email: new Control('', Validators.compose([
      Validators.required,
      validateEmail
    ]))
  });
}
{% endraw %}
{% endhighlight %}

## Building custom validator directives

Now that we're able to add our custom validator to our form controls imperatively when building model-driven forms, we might also enable our validator to be used in template driven forms. In other words: We need a directive. The validator should be usable like this:

{% highlight html %}
{% raw %}
<form novalidate>
  ...
  <input type="email" ngControl="email" validateEmail>
</form>
{% endraw %}
{% endhighlight %}

`validateEmail` is applied as an attribute to the `<input>` DOM element, which already gives us an idea what we need to do. We need to build a directive with a matching selector so it will be executed on all input controls where the directive is applied. Let's start off with that.

{% highlight js %}
{% raw %}
import {Directive} from '@angular/core';

@Directive({
  selector: '[validateEmail][ngControl]'
})
class EmailValidator {}
{% endraw %}
{% endhighlight %}

We import the `@Directive` decorator form `angular2/core` and use it on a new `EmailValidator` class. If you're familiar with the `@Component()` decorator that this is probably not new to you. In fact, `@Directive` is a superset of `@Component` which is why most of the configuration properties are available.

Okay, technically we could already make this directive execute in our app, all we need to do is to add it as a `directives` dependency to the component where it's used:


{% highlight js %}
{% raw %}
@Component({
  ...
  directives: [EmailValidator]
})
class Cmp {
  ...
}
{% endraw %}
{% endhighlight %}

Even though this works, there's nothing our directive does at the moment. What we want to do is to make sure that our custom validator is executed when Angular compiles this directive. How do we get there?

Angular has an internal mechanism to execute validators on a form control. It maintains a **multi provider** for a dependency token called `NG_VALIDATORS`. If you've read our article on [multi providers in Angular 2](http://blog.thoughtram.io/angular2/2015/11/23/multi-providers-in-angular-2.html), you know that Angular injects multiple values for a single token that is used for a multi provider. If you haven't, we highly recommend checking it out as the rest of this article is based on it.

It turns out that **all** built-in validators are already added to the `NG_VALIDATORS` token. So whenever Angular instantiates a form control and performs validation, it basically injects the dependency for the `NG_VALIDATORS` token, which is a list of all validators, and executes them one by one on that form control.

Since multi providers can be extended by adding more multi providers to a token, we can consider `NG_VALIDATORS` as a hook to add our own validators.

Let's add our validator to the `NG_VALIDATORS` via our new directive:

{% highlight js %}
{% raw %}
import {Directive, provide} from '@angular/core';
import {NG_VALIDATORS} from '@angular/common';

@Directive({
  selector: '[validateEmail][ngControl]',
  providers: [
    provide(NG_VALIDATORS, {
      useValue: validateEmail,
      multi: true
    })
  ]
})
class EmailValidator {}
{% endraw %}
{% endhighlight %}

Again, if you've read our article on multi providers, this should look very familiar to you. We basically add a new value to the `NG_VALIDATORS` token by taking advantage of multi providers. Angular will pick our validator up by injecting what it gets for `NG_VALIDATORS`, and performs validation on a form control. Awesome, we can now use our validator for model-driven **and** for template-driven forms!

## Custom Validators with dependencies

Sometimes, a custom validator has dependencies so we need a way to inject them. Let's say our email validator needs an `EmailBlackList` service, to check if the given control value is not only a valid email address but also not on our email black list (in an ideal world, we'd build a separate validator for checking against an email black list, but we use that as a motivation for now to have a dependency).

**The not-so-nice way**

One way to handle this is to create a factory function that returns our `validateEmail` function, which then uses an instance of `EmailBlackList` service. Here's what such a factory function could look like:

{% highlight js %}
{% raw %}
import {Control} from '@angular/common';

function validateEmailFactory(emailBlackList: EmailBlackList) {
  return (c: Control) => {
    let EMAIL_REGEXP = ...

    let isValid = /* check validation with emailBlackList */

    return isValid ? null : {
      validateEmail: {
        valid: false
      }
    };
  };
}
{% endraw %}
{% endhighlight %}

This would allow us to register our custom validator via dependency injection like this:

{% highlight js %}
{% raw %}
@Directive({
  ...
  providers: [
    provide(NG_VALIDATORS, {
      useFactory: (emailBlackList) => {
        return validateEmailFactory(emailBlackList);
      },
      deps: [EmailBlackList]
      multi: true
    })
  ]
})
class EmailValidator {}
{% endraw %}
{% endhighlight %}

We can't use `useValue` as provider recipe anymore, because we don't want to return the factory function, but rather what the factory function **returns**. And since our factory function has a dependency itself, we need to have access to dependency tokens, which is why we use `useFactory` and `deps`. If this is entirely new to you, you might want to read our article on [Dependency Injection in Angular 2](http://blog.thoughtram.io/angular/2015/05/18/dependency-injection-in-angular-2.html) before we move on.

Even though this would work, it's quite a lot of work and also very verbose. We can do better here.

**The better way**

Wouldn't it be nice if we could use constructor injection as we're used to it in Angular 2? Yes, and guess what, Angular has us covered. It turns out that a validator can also be a **class** as long as it implements a `validate(c: Control)` method. Why is that nice? Well, we can inject our dependency using constructor injection and don't have to setup a provider factory as we did before.

Here's what our `EmailValidator` class would look like when we apply this pattern to it:

{% highlight js %}
{% raw %}
@Directive({
  ...
})
class EmailValidator {
  
  validator: Function;

  constructor(emailBlackList: EmailBlackList) {
    this.validator = validateEmailFactory(emailBlackList);
  }

  validate(c: Control) {
    return this.validator(c);
  }
}
{% endraw %}
{% endhighlight %}

However, we now need to adjust the provider for `NG_VALIDATORS`, because we want to use an instance of `EmailValidator` to be used for validation, not the factory function. This seems easy to fix, because we know that we create instances of classes for dependency injection using the `useClass` recipe.

However, we already added `EmailValidator` to the `directives` property of our component, which is a provider with the `useClass` recipe. We want to make sure that we get the exact same instance of `EmailValidator` on our form control, even though, we define a new provider for it. Luckily we have the `useExisting` recipe for that. `useExisting` defines an alias token for but returns the same instance as the original token:

{% highlight js %}
{% raw %}
@Directive({
  ...
  providers: [
    provide(NG_VALIDATORS, {
      useExisting: EmailValidator,
      multi: true
    })
  ]
})
class EmailValidator {
  ...
}
{% endraw %}
{% endhighlight %}

**Yikes! This won't work** . We're referencing a token (`EmailValidator`) which is undefined at the point we're using it because the class definition itself happens later in the code. That's where `forwardRef()` comes into play.

{% highlight js %}
{% raw %}
import {forwardRef} from '@angular/core';

@Directive({
  ...
  providers: [
    provide(NG_VALIDATORS, {
      useExisting: forwardRef(() => EmailValidator),
      multi: true
    })
  ]
})
class EmailValidator {
  ...
}
{% endraw %}
{% endhighlight %}

If you don't know what `forwardRef()` does, you might want to read our article on [Forward References in Angular 2](http://blog.thoughtram.io/angular/2015/09/03/forward-references-in-angular-2.html).

Here's the full code for our custom email validator:

{% highlight js %}
{% raw %}
import {provide, Directive, forwardRef} from '@angular/core';
import {NG_VALIDATORS, Control} from '@angular/common';

function validateEmailFactory(emailBlackList: EmailBlackList) {
  return (c: Control) => {
    let EMAIL_REGEXP = /^[a-z0-9!#$%&'*+\/=?^_`{|}~.-]+@[a-z0-9]([a-z0-9-]*[a-z0-9])?(\.[a-z0-9]([a-z0-9-]*[a-z0-9])?)*$/i;

    return EMAIL_REGEXP.test(c.value) ? null : {
      validateEmail: {
        valid: false
      }
    };
  };
}

@Directive({
  selector: '[validateEmail][ngControl],[validateEmail][ngModel],[validateEmail][ngFormControl]',
  providers: [
    provide(NG_VALIDATORS, {
      useExisting: forwardRef(() => EmailValidator),
      multi: true
    })
  ]
})
export class EmailValidator {

  validator: Function;

  constructor(emailBlackList: EmailBlackList) {
    this.validator = validateEmailFactory(emailBlackList);
  }

  validate(c: Control) {
    return this.validator(c);
  }
}
{% endraw %}
{% endhighlight %}

You might notice that we've extended the selector, so that our validator not only works with `ngControl` but also with `ngModel` and `ngFormControl` directives. If you're interested in more articles on forms in Angular 2, make sure to subscribe to our blog as we have more in the pipe which will be published soon!
