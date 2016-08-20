+++
Description = "Neat errors in JavaScript"
Tags = ["javascript", "es6", "es2015", "errors"]
date = "2016-08-20T07:50:35-04:00"
title = "Error is for everyone"

+++

Writing Go has really taught me good manners in regards to error handling.
Idiomatic Go handles errors right as they happen. 

This got me thinking today. I was working on an AngularJS app and I
thought it would be great to just pass `errors.New` onto the next
stage of a Promise. It was then that I realized I could actually
define a custom error type for use in my application, just like I
could in Go.

The [spec][1] defines Error objects.

> Instances of Error objects are thrown as exceptions when runtime
errors occur. The Error objects may also serve as base objects for
user-defined exception classes.

The key bit is that last sentence, which allows us to define
our own Error objects.

```
// App Error is used for handling errors specific to our
// problem domain.
function appError(msg) {
    const err = Error(msg);

    err.name = "App Error"

    return err;
}
```

Since I’m working with Angular, I created a factory for my
custom Error and called it `appError`.

```
function factory {
    function appError(msg) {
        const err = Error(msg);

        err.name = "App Error";

        return err;
    }

    return appError;
}

angular.module('myApp', []).factory('appError', factory);
```

Now I can include the appError factory wherever I need it and have a
custom error type for my app. I find this especially useful when
combined with $log. I use $log to send logs to Loggly, and I can
see my custom App Errors right in the console.

[1]: http://www.ecma-international.org/ecma-262/6.0/#sec-error-objects