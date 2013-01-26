# Tutorial 13 - Don't Repeat Yourself

In the [latest tutorial][1] we implemented a very rude server-side
validator for the `loginForm` which exercises the same kind of
syntacic rules previously implemented on the client-side.

One of our long term objectives is to eliminate any code duplication
from our web applications.  That's like to say to firmly stay as compliant as we
can with the Don't Repeat Yourself (DRY) principle.

# Introduction

In this tutorial of the *modern-cljs* series we're going to respect the
DRY principle in the mean time we adhere to the progressive enhancement
strategy. Specifically, we'll try to exercise both the principle and the
strategy in one of the most relevant context in developing a web
application: the form fields validation.

Let's start by writing down few technical intermediate requirements to
solve this not so easy problem space. We need to:

* select a good server-side validator library
* verify its portability from CLJ to CLJS
* port the library from CLJ to CLJS
* define a set of validators to be shared between the server and the
  client code
* exercise the defined validators on the server and cliente code.

That's a lot of work to be done for a single tutorial. So take your time
to follow it step by step.

# The selection process

If you search [Github][] for a CLJ validator library you'll find quite a
large number of results, but if you restrict the search to CLJS
validator library only, you currently get just one lib: [valip][]. Valip
is a port on CLJS of the [original CLJ valip][]. This is already a good
result by itself, becasue it demonstrates that we share our long term
objective with someone else. If you then take a look at the owners of
those two Github repos, you'll discover that they are two of the most
prolific and active clojure-ists: [Chas Emerick][] and
[James Reeves][]. I'd love that the motto *Smart people think alike* was
true.

We will eventually search for others CLJ validator libraries later. For
the moment, by following the Keep It Small and Stupid (KISS) pragmatic
approach, we stay with the Valip library which already seems to satify
the first two intermediate requirements we just listed in the
introduction: its goodness should be guaranteed by the quality of its
owners and it runs on both CLJ and CLJS.

# The server side validation

Let's start by using Valip on the server-side first. Valip usage is damn
simple and well documented in the [readme][] file which I encourage you
to read.

First, Valip provides you a `validate` function from the `valip.core`
namespace. It accepts a map (i.e key/value pairs) and one or more
vectors. Each vector consists of one of map keys, a predicate function and an
error string, like so:

```clojure
(validate map-of-values
  [key-1 predicate-1 error-1]
  [key-2 predicate-2 error-2]
  ...
  [key-n predicate-n error-n])
```

To keep things simple, we are going to apply the Valip lib to our old
`Login` tutorial sample. Here is a possible `validate` usage for the
`loginForm` input elements:

```clojure
(validate {:email email :password password}
  [:email present? "Email can't be empty"]
  [:email email-address? "Invalid email format"]
  [:password present? "Password can't be empty"]
  [:password (matches *re-password*) "Invalid email format"])
```

As you can see you can attach more one or more predicate/function to the
same key.  If no predicate fails, `nil` is returned. If at least one
predicate fails, a map of keys to error values is returned. Again, to
make things easier to be understood, suppose for a moment that the
`email` value passed to `validate` was not well formed and that the
`password` was empty. You would get a result like the following:

```clojure
{:email ["Invalid email format"]
 :password ["Password can't be empty" "Invalid password format"]}
```

The value of each key of an error map is a vector, because the
`validate` function can catch for more than one predicate failure for
each key. That's a very nice feature to have.

## User defined predicates and functions

Even if Valip library already provides, via the `valip.predicates`
namespace, a relatively wide range of pre-defined predicates and
functions returning predicates, at some point you'll need to define new
predicates by yourself. Valip lets you do this very easily too.

A predicate accepts a single argument and returns `true` or `false`. A
function returning a predicate accepts a single argument and returns a
function which accept a single argument and returns `true` or
`false`.

Nothing new for any clojure-ist which knows about HOF (Higher Order
Function), but another nice feature to have in the hands.

If you need to take some inspiration, here are the definitions of the
built-in `presence?` predicate and the `matches` function which returns a
predicate.

```clojure
;;; from valip.predictes namespace

;;; present? predicate
(defn present?
  [x]
  (not (string/blank? x)))

;;; matches function
(defn matches
  [re]
  (fn [s] (boolean (re-matches re s))))
```

## A doubt to be cleared

If there is a thing that it does not intrigue me about the original
Valip library is its dependency from a lot of java packages in the
`valip.predicates` namespace.

```clojure
(ns valip.predicates
  "Predicates useful for validating input strings, such as ones from a HTML
  form."
  (:require [clojure.string :as string]
            [clj-time.format :as time-format])
  (:import
    [java.net URL MalformedURLException]
    java.util.Hashtable
    javax.naming.NamingException
    javax.naming.directory.InitialDirContext
    [org.apache.commons.validator.routines IntegerValidator
                                           DoubleValidator]))
```

This is not a suprise if you take into account that it has been created
more that two years ago, when CLJS was eventually floating just in few
clojure-ist minds.

The surprise is that Chas Emerick has choosen it just five months ago,
when CLJS was already been reified from a very smart idea into a
programming language. So, if the original valip library was so dependent
on the JVM, why Chas Cemerick choose it when he decided to port a
validator from CLJ to CLJS instead of choofing a CLJ validator less
compromised with the underlaying JVM? I don't know. Maybe the answer is
just that the predefined predicates and functions were confined in the
`valip.predicates` namespace and most of them were easily redefinable in
terms of JS interop features of CLJS.

## First try

We already talked to much. Let's see the `valip` lib at work by
validating our old `loginForm` friend.

First you have to add the forked `valip` library to the project
dependencies.

```clojure
;;; code fragment from project.clj
  :dependencies [[org.clojure/clojure "1.4.0"]
                 [compojure "1.1.5"]
                 [hiccups "0.2.0"]
                 [com.cemerick/shoreleave-remote-ring "0.0.2"]
                 [shoreleave/shoreleave-remote "0.2.2"]
                 [com.cemerick/valip "0.3.2"]]
```

To follow at our best the principle of separation of concerns, let's
create a new namespace specifically dedicated to the `loginForm` fields
validations. Create the `login` subdirectory under the
`src/clj/modern_cljs/` directory and then create a new file named
`validators.clj`. Open it and declare the new
`modern-cljs.login.validators` namespace by requiring all the namespaces
you need.

```clojure
(ns modern-cljs.login.validators
  (:require [valip.core :refer [validate]]
            [valip.predicates :refer [present? matches email-address?]]))
```

In the same file you now have to define the validators for the user
credential (i.e `email` and `password`).

```clojure
(def ^:dynamic *re-password* #"^(?=.*\d).{4,8}$")

(defn validate-user-credential [email password]
  (validate {:email email :password password}
            [:email present? "Email can't be empty."]
            [:email email-address? "The provided email is invalid."]
            [:password present? "Password can't be empty."]
            [:password (matches *re-password*) "The provided password is invalid"]))
```

> NOTE 1: Again, to follow the separation of concerns principle, we
> moved here the *re-password* regular expression from the `login.clj`.

> NOTE 2: Valip provides the `email-address?` built-in predicate which
> matches the passed email value against an embedded regular
> expression. This regular expression is based on RFC 2822 and it is
> defined in the `valip.predicates` namespace. This is why the
> `*re-email*` regular expression is not needed anymore.

> NOTE 3: Valip also provides the built-in `valid-domain-email?` predicate
> which, by running a DNS lookup, verify even the validity of the email
> domain.

We know need to update the `login.clj` by calling the just defined
`validate-user-credential` function.  Open the `login.clj` file from the
`src/clj/modern_cljs/` directory and modify it as follows:

```clojure
(ns modern-cljs.login
  (:require [modern-cljs.login.validators :refer [validate-user-credential]]))

(defn authenticate-user [email password]
  (let [{email-messages :email
         password-messages :password} (validate-user-credential email password)]
    (if (and (empty? email-messages)
             (empty? password-messages))
      (str email " and " password
           " passed the formal validation, but we still have to authenticate you")
      (str "Please complete the form."))))
```

> NOTE 4: Even if we could have returned back to the user more detailed
> messages from the validator result, to maintain the same behaviour of
> the previous version we only return the *Please complete the form.*
> message when the user typed something wrong in the form fields.

I don't know about you, but even for such a small case, the use of a
validator library seems to me to be effective, at least in terms of
clarity of the code.

Anyway, let's run the application as usual to verify that the
interaction with the just added validator is still working as expected,
that means as at the end of the [latest tutorial][].

```clojure
$ rm -rf out # it's better to be safe than sorry
$ lein clean # it's better to be safe than sorry
$ lein cljsbuild clean # it's better to be safe than sorry
$ lein cljsbuild auto dev
$ lein ring server-headless # in a new terminal
```

Repeat all the interactiion tests we executed in the
[latest tutorial][]. Remeber to first disabled the JS of the
browser.

When you submit the [login-dbg.html][] page you should receive a `Please
complete the form` message anytime you do not provide the `email` and/or
the `password` values and anytime the provided values do not pass the
corresponding validation predicates.

When the provided values for the email and password input elements pass
the validator rules, you should receive the following message:
*xxx.yyy@gmail.com and zzzx1 passed the formal validation, but we still
have to authenticate you*. So far, so good.

It's now time to take seriusly Chas Emerick and try to use the `valip`
lib on the client-side.

# Cross the border

As you can read from the [readme][] file of the Chas Cemerick's
[valip fork][], he tried to make its fork as portable as possibile
between CLJ and CLJS and, as we were supposing at the beginning of this
tutorial, most of the differences reside in the `valip.predicates`
namespace which, as we said, originally required a lot of java packages.

You can find the portable predicates in the `valip.predicates` namespace
we already used. The platform-specifc predicates can be found in
`valip.java.predicates` and `valip.js.predicates`.

## Hard time

It's now time to dedicate our attention to the client-side validation to
verify that we can use the [valip fork][] from Chas Emerick.

Before doing any code modification let's face one big problem: the so
called [Feature Expression Problem][].

> Writing programs that target Clojure and ClojureScript involves a lot of
> copy and pasting. The usual approach is to copy the whole code of one
> implementation to a source file of the other implementation and to
> modify the platform dependent forms until they work on the other
> platform. Depending on the kind of program the platform specific code is
> often a fraction of the code that works on both platforms. A change to
> platform independent code requires a modification of two source files
> that have to be kept in sync. To solve this problem branching by target
> platform on a form level would help a lot.

The current workaround for this really big problem is to ask the help of
the [lein-cljsbuild][] plugin we have been already using from the very
beginning of this series of short tutorials on ClojureScript.

## Thanks Evan Mezeske

[Evan Mezaske][] mada a great job by donating `lein-cljsbuild` to the
clojurescript community. I suspect that without it I would never been
able even to launch the CLJS compiler. Big, really big thanks to him and
to all the others who helps him to keep it updated with the frequently
release of any new CLJS and Google Closure Compiler releases.

Lein-cljsbuild plugin has a `:crossovers` option which allows to CLJ and
CLJS to share any code that is not specific to the CLJ/CLJS underlying
virtual machines (i.e. JVM and JSVM).

The trick is pretty simple. Any namespace you want to share between CLJ
and CLJS has to be put in a vector which is attached to the
`:crossovers` keyword option of the `:cljsbuild` section of your
project. It's quicker to do than it's to say. Here is the interested
`project.clj` code fragment.

```clojure
  ;; code fragment from project.clj

  ;; cljsbuild tasks configuration
  :cljsbuild {:crossovers [valip.core valip.predicates modern-cljs.login.validators]
              :builds
              {:dev
               {;; clojurescript source code path
                :source-path "src/cljs"

                ;; Google Closure Compiler options
                :compiler {;; the name of emitted JS script file
                           :output-to "resources/public/js/modern_dbg.js"

                           ;; minimum optimization
                           :optimizations :whitespace
                           ;; prettyfying emitted JS
                           :pretty-print true}}
  ;; other code follows
```

As you can seed we added three namesspaces to the `:crossovers` option:

* `valip.core` namespace which includes the portable CLJ/CLJS core code
  of the `valip` library;
* `valip.predicates` namespace which includes the portable CLJ/CLJS code
  where the portable `predicates` are defined;
* `modern-cljs.login.validators` namespace which includes the validator
  we defined and that we want to share between the server-side and
  client-side of our web application.

To have a much better understanding of the `:crossovers` option I
strongly recommend you to read the [original documentation][].

## Don't Repeat Yourself

Here we are. We reached the point. Let's see it the magic works.

Open the `login.cljs` file from the `src/cljs/modern-cljs/` directory
and start by first adding the `modern-cljs.login.validators` namespace
where we defined the `validate-user-credential` validator.

```clojure
(ns modern-cljs.login
  (:require-macros [hiccups.core :refer [html]])
  (:require [domina :refer [by-id by-class value append! prepend! destroy! attr log]]
            [domina.events :refer [listen! prevent-default]]
            [hiccups.runtime :as hiccupsrt]
            [modern-cljs.login.validators :refer [validate-user-credential]]))
```


If we want to preserve some correspondance between the CLJS and CLJ
validation code we have to refactor the former. the First step is to
copy the `validate-email` validator definition from `login.clj` file to
`login.cljs` file.

```clojure
(defn validate-email [email]
  (validate {:email email}
            [:email present? "Email can't be empty"]
            [:email email-address? "The provided email is invalid"]))
```

As you remember we [previously][] got the email regular expresssion from
the `pattern` attribute of the `email` element of the form. Now we're
using the valip `email-address?` predicate.  It embeds a different and
more powerful regular espression which is compliant with the RFC
2822. We should update the `pattern` attribute of the `email` element
with the same regular-expression used by the `email-address?`
predicate. Do you see that we are violating two times the *Don't Repeat
Yourself* (DRY) principle? One by copying and pasting the email
validator from `login.clj` to `login.cljs`. And a second time by copying
the regular expression embedded in the `email-address?` predicate and
pasting it into the `pattern` attribute configured in the `email`
element of the `loginForm` to enable the new HTML5 validation
features. Too bad.

Anyway, for the moment be forgiving with ourself and go ahead with the
CLJS validation code by copying and pasting the `validate-password`
definition from `login.clj` to `login.cljs`. This time we have to update
our code by getting the password regular expression from the `pattern`
attribute of the `password` element of the `loginForm`.  Other two
violations of the DRY principles. Too, too bad. Be forgiving again and
go ahead.

```clojure
(defn validate-password [password]
  (validate {:password password}
            [:password present? "Password can't be empty"]
            [:password (matches (attr password :pattern) "The provided password is invalid"]))
```

It's now time to use the `validate-email` and `validate-password`
results to manipulate the `loginForm` DOM for helping the user to avoid
typos in filling out the form fields.

```clojure
(defn email-validator [email]
  (destroy! (by-class "email"))
  (if-let [errors (validate-email email)]
    (do
      (prepend! (by-id "loginForm") (html [:div.help.email (first (:email errors))]))
      false)
    true))

(defn password-validator [password]
  (destroy! (by-class "password"))
  (if-let [errors (validate-password password)]
    (do
      (append! (by-id "loginForm") (html [:div.help.password (first (:password errors))]))
      false)
    true))
```

> NOTE 4: We now used the `if-let` form instead of the `if` form because
> we need to use the `if` condition result in the `true` branch.

> NOTE 5: Take a look at how we got the error message to be shown to the
> user from the `validate-email` and the `validate-password` functions.

We just updated the `blur` event handlers. Now we need to update the `click`
event hanlder of the `submit` button.

```clojure
(defn validate-user [email password]
  (validate {:email email :password password}
            [:email present? "Email can't be empty"]
            [:email email-address? "The provided email is invalid"]
            [:password present? "Password can't be empty"]
            [:password (matches (attr password :pattern)) "The provided password is invalid"]))
```

> NOTE 6: Here we used a more sophisticated valip `validate` form. The one
> in wich you can manage more element values to be validated (i.e. `email`
> and `password`) and more validator for each value (i.e. `present?` and
> `email-address?` for the `email` value and `present?` and `matches` for
> the `password` value.

> NOTE 7: We repeated ourself again by using in the `validate-user`
> function the same validators we already used for `validate-email` and
> `validate-password. For the moment keep on being forgiving again.

We still have to modify the `validate-form` handler by using the just
defined `validate-user` and the `init` function by using the new
`email-validator` and `password-validator` functions. We refactor a lot
of code until now without testing anything. For the moment, to verify
that everything is still working, we limit ourself to substitute the
current calls to the old `validate-email` and `validate-password` with
the new `email-validator` and `passsord-validator`.

```clojure
(defn validate-form [evt]
  (let [email (by-id "email")
        password (by-id "password")
        email-val (value email)
        password-val (value password)]
    (if (or (empty? email-val) (empty? password-val))
      (do
        (destroy! (by-class "help"))
        (prevent-default evt)
        (append! (by-id "loginForm") (html [:div.help "Please complete the form"])))
      (if (and (email-validator email)
               (password-validator password))
        true
        (prevent-default evt)))))

(defn ^:export init []
  (if (and js/document
           (aget js/document "getElementById"))
    (let [email (by-id "email")
          password (by-id "password")]
      (listen! (by-id "submit") :click (fn [evt] (validate-form evt)))
      (listen! email :blur (fn [evt] (email-validator email)))
      (listen! password :blur (fn [evt] (password-validator password))))))
```




Before to commit ourself with `valip` lib, let's see at least one
alternative: [metis][].

The metis validation lib is very young and is inspired to the ruby
Active Records Validation approach. Metis defines the macro
`defvalidator` which expands in a validator function. We know about
macros limitation in JS, but we also know how they can be easly
managed by isolating them and then by excplicitly requiring their
namespace when needed. So, it should not be a PITA to port metis on
CLJS.

Anyway, let's first take a look at the metis usage before talking
about its porting on CLJS just to verify if satisfy our requirement of
semplicity and completeness by writing down some usage samples:

```closure
;;; email value is not nil
(defvalidator email-validator
        [:email :presence])

;;; email values is not nil and add an optional error message
(defvalidator email-validator
        [:email :presence {:message "Please enter your email address"}])

;;; email and password are not nil with a common error message
(defvalidator user-validator
        [[:email :password] {:message "This field is required"}])

;;; email presence and pattern
(defvalidator email-validator
        [:email [:presence :formatted {:pattern #"a pattern"}]])

;;; email and password togheter


Let's start by adding in our project the server-side valip lib.

```clojure
;;; project.clj code fragment

  :dependencies [[org.clojure/clojure "1.4.0"]
                 [compojure "1.1.5"]
                 [hiccups "0.2.0"]
                 [com.cemerick/shoreleave-remote-ring "0.0.2"]
                 [shoreleave/shoreleave-remote "0.2.2"]
                 [valip "0.2.0"]]
```



```
# Next step - TBD

TBD

# License

Copyright © Mimmo Cosenza, 2012-13. Released under the Eclipse Public
License, the same as Clojure.

[1]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-12.md
[]: https://github.com/cemerick
[]: https://github.com/weavejester
[]: https://github.com/weavejester/valip#usage
[]: https://github.com/mylesmegyesi/metis
[]: https://github.com/cemerick/valip/blob/master/README.md
[]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-12.md#first-try
[]: http:/localhost:3000/login-dbg.html
[]: https://github.com/magomimmo/modern-cljs/blob/master/doc/tutorial-12.md#the-highest-surface
[]: http://dev.clojure.org/display/design/Feature+Expressions
[]: https://github.com/emezeske/lein-cljsbuild
[]: https://github.com/emezeske/lein-cljsbuild/blob/master/doc/CROSSOVERS.md#sharing-code-between-clojure-and-clojurescript
[]: https://github.com/emezeske
[]: https://github.com/emezeske/lein-cljsbuild/blob/0.3.0/doc/CROSSOVERS.md