# firemore

**Firemore** is a library for writing [clojure](https://clojure.org/)([script](https://clojurescript.org/)) applications using [Google Firestore](https://cloud.google.com/firestore).

Main features include:
1. A direct API to the [Firestore Database](https://firebase.google.com/docs/firestore).
1. Automatic (and customizable) conversions between clojure maps and Firestore "json" documents.
1. A channels based API for getting and observing Firestore documents.
1. A binding between the Firestore Cloud database and a local clojure atom (great for om/re-frame/reagent).

# Table of Contents
1. [Getting Started](#getting_started)
1. [Usage](#usage)
1. [API](#api)
1. [Contributing](#contributing)
1. [Credits](#credits)
1. [License](#license)

# <a id="getting_started"></a>Getting Started

To use firemore in an existing project, simply add this to your dependencies in project.clj ([lein](https://github.com/technomancy/leiningen)) or build.boot ([boot](https://github.com/boot-clj/boot)).

!!!!! [firemore "x.y.z"] @clojars !!!!!!

# <a id="usage"></a>Usage

;; TODO: Transparently convert arrays to Sets and back. Auto index for containment?

;; TODO: Transparently convert keywords to strings and back? Maybe write some checksum at front of keyword?

;; TODO: Add in metadata whether value you see if realized or not.

The following is a walkthrough of the main features of firemore.

## References

Read the documentation on [documents, collections,
and references](https://firebase.google.com/docs/firestore/data-model) (just
the linked page). Go ahead. I'll wait.

As you just read, a Firestore reference is a opaque javascript object with a
bunch of functions attached to it. In firemore a reference is a vector of
keywords or strings with length at least 1.

So, the following document reference in firestore
```javascript
db.collection('users').doc('alovelace');
```
Becomes this in firemore:
```clojure
["users" "alovelace"] ;; OR
[:users  "alovelace"]  ;; OR
["users" :alovelace]  ;; OR
[:users  :alovelace]
```

Note that keywords and strings are interchangeable. I prefer to use keywords
in collection (odd) positions and strings in (even) positions, but it is up to
you.

Note that a vector of even length must be a reference to a
document, while a vector of odd length must be a reference to a collection. So
`[:users]` is a reference to the `users` collection. While
`[:users "alovelace"]` is a reference to a document *within* the `users`
collection.

## Read from Firestore

The map to your right shows the data currently in your section of the firestore
database. I have taken the liberty of setting you up with some starting data so
that we can practice reading it using firemore.

```clojure
(def user-atm (get-user-atom))

(def user-id (:uid @user-atm))

(def luke-reference [:users user-id :characters "luke"])
```

Note that the reference begins with `[:users user-id]`. This is because
I have set up [security rules](https://firebase.google.com/docs/firestore/security/get-started)
so you and only you may read and write to a location
under `users/<user-id>` in the Firestore database. This is necessary because
this database is being used by everyone currently looking at this documentation. The
security rule allows me to carve out a little place in the database for you to
play without conflicting with others.

Is Luke Skywalker a force user? I can never remember? Let's check!
```clojure
(go
  (let [luke (<! (get luke-reference))]

  (println "luke ->" luke)

  (println "Is luke a force user? " (:force-user? luke))))
```

That's right, he does have force powers! Couldn't remember.

A firemore document is a regular [clojure map](https://clojure.org/reference/data_structures#Maps).
There is a fair amount of code to allow for conversion between a Firestore
document and a firemore document (see [Clojure Interop](#clojure_interop) for details).

```
Usage:
(get reference)

Returns a channel. If a document exist at reference, it will be put! upon the
channel. If no document exist at reference, then :firemore/no-document will be
put! on the channel. The channel will then be closed.

Note:
put! ->  clojure.core.async/put!
```

But what if I want to watch Luke change over time? I could periodically check for
updates; this is error prone, verbose, and inefficient. Rather than getting
Luke once, let's watch him from now on.

```clojure
(async/go
  (let [luke-chan (watch luke-reference)
        initial-luke (<! luke-chan)]
    (println "Initial Luke ->" initial-luke)

    (println "Merging in Luke's teenage occupation...")
    (merge! luke-reference {:occupation "farmboy"})
    (println "teenage Luke ->" (<! luke-chan))

    (println "Changing Luke's adult occupation...")
    (write! luke-reference (assoc initial-luke :occupation "jedi"))
    (println "adult Luke ->" (<! luke-chan))

    (println "Changing to Lukes final occupation after episode 7")
    (write! luke-reference (assoc initial-luke :occupation "One with the Force"}))
    (println "Episode 7 Luke ->" (<! luke-chan))

    (println "Remove luke from the Firestore database")
    (delete! luke-reference)
    (println "Removed Luke reference ->" (<! luke-chan))

    ;; Remember to close the channel when you are done with it!
    (close! luke-chan)))
```

```
Usage:
(watch reference)

Returns a channel. If a document exist at reference, it will be put! upon
the channel. If no document exist at reference, then :firemore/no-document will
be put! on the channel. As the document at reference is updated through
time, the channel will put! the newest value of the document (if it exist)
or :firemore/no-document (if it does not) upon the channel.

Important: close! the channel to clean up the state machine feeding this
channel. Failure to close the channel will result in a memory leak.

Note:
put! ->  clojure.core.async/put!
close! ->  clojure.core.async/close!
```

## Write to Firestore

```
Usage:
(write! reference m)

Returns a channel. Overwrites the document at reference with m.  Iff an error
occurs when writing m to Firestore, then the error will be put! upon the
channel. The channel will then be closed.

Note:
put! ->  clojure.core.async/put!
```

```
Usage:
(merge! reference m)

Returns a channel. Updates (merges in novelty) the document at reference with m.
Iff an error occurs when writing m to Firestore, then the error will be put!
upon the channel. The channel will then be closed.

Note:
put! ->  clojure.core.async/put!
```

```
Usage:
(delete! reference)

Returns a channel. Iff an error occurs when deleting reference from Firestore,
then the error will be put! upon the channel. The channel will then be closed.

Note:
put! -> clojure.core.async/put!
```

## Queries

First [read the documentation on queries](https://firebase.google.com/docs/firestore/query-data/queries). As you just read, in Firestore queries are built from a collection reference. In
firemore queries are built by adding a query map to the end of the reference
vector.

So this in Firestore
```javascript
db.collection("cities").where("state", "==", "CA").where("population", "<", 1000000);
```

becomes this in firemore
```clojure
[:cities {:where [["state" "==" "CA"]
                  ["population" "<" 1000000]]}]
```

Queries also support the [orderBy and limit option](https://firebase.google.com/docs/firestore/query-data/order-limit-data).

So this in Firestore
```javascript
citiesRef.where("population", ">", 100000).orderBy("population").orderBy("state", "desc").limit(2)
```

becomes this in firemore
```clojure
[:cities {:where [["population" "<" 1000000]]
          :order [["population" "asc"] ["state" "desc"]]
          :limit 2}]
```

If you have only one `:where` clause predicate, it is fine to specify it as a
single vector. So this is also equivalent to the above.

```clojure
[:cities {:where ["population" "<" 1000000]
          :order [["population" "asc"] ["state" "desc"]]
          :limit 2}]
```

In a similar fashion, the `:order` values are expanded into 2 element
vectors of `[<property> "asc"]` if they are specified as strings. So the following
is also equivalent to the above.

```clojure
[:cities {:where ["population" "<" 1000000]
          :order ["population" ["state" "desc"]]
          :limit 2}]
```

Finally, the `:start-at`, `:start-after`, and `end-at` options from [paginate data with query-cursors](https://firebase.google.com/docs/firestore/query-data/query-cursors)
are also supported as options.

## Build Local State Atom

Let's say you needed to keep track of the best of the [Three Stooges](https://en.wikipedia.org/wiki/The_Three_Stooges)? How might you go about doing this?

```HTML
The best stooge is <span id="best-stooge-value"></span>.

<button id="moe-button"></button>
<button id="larry-button"></button>
<button id="curly-button"></button>
```

```clojure
(def best-stooge-reference [:users user-id :best "stooge"])

(def best-stooge-chan (watch best-stooge-reference))

(go-loop []
  (when-let [{:keys [value]} (<! best-stooge-chan)]
    (reset! best-stooge value)
    (aset! (js/document.querySelector "#best-stooge-value") "value" value)
    (recur)))

(doseq [stooge ["moe" "larry" "curly"]]
  (aset! (js/document.querySelector (str "#" stooge "-button"))
         "onclick"
         (fn [event] (merge! counter-reference {:value stooge}))))
```

The following code does works. However, it is an annoyance to have to create a channel from a reference, create a `go-loop` that consumes from said channel, and close the channel in order to properly dispose the `go-loop`.

Firemore has a solution to that.

```clojure
(def atm (atom {:paths {[:friends] [:app user-id :friends]
                        [:enemies] [:app user-id :enemies]
                        [:favorite :dog] [:app user-id :favorite "dog"]
                        [:favorite :cat] [:app user-id :favorite "cat"]}}))

;; Allows us to see the changes to the atom over time
(add-watch atm :print-firestore-changes
  (fn [_ _ old new]
    (let [only-in-old only-in-new _] (clojure.data/diff old new)]
      (when-not (empty? only-in-old)
        (println "Removed:")
        (doseq [o only-in-old]
          (println "-" o)))
      (when-not (empty? only-in-new)
        (println "Added:")
        (doseq [n only-in-new]
          (println "-" n))))))

;; Updates figure-1 to the newest value within atm
(add-watch atm :display-atom
  (fn [_ _ _ new] (display-atom "figure-1" new)))

(realize atm)
```

As you can see in figure-1, all of the path/reference (key/value) pairs within `:paths` have become realized things within a map in `:firestore`. If you remove a key from `:paths` it will remove the same path from within `:firestore`. Similarly if you add a new path/reference to `:paths` it will add a corresponding location in `:firestore`.

## Authentication

Most apps require some way of authenticating a user.
Firestore includes a fairly robust [Authentication System](https://firebase.google.com/docs/auth).
Use of the built in authentication system will allow you to complete your project
more quickly and securely than rolling your own solutions.

```clojure
(def user-atm (get-user-atom))

;; The value within user-atm is currently nil as you are not logged in.
(assert (= nil @user-atm))

;; Let's log you in as the anonymous user
(login-as-anonymous)

;; having done that, let's check what user-atm looks like now
(println "(1) user-atm contains -> " @user-atm)

;; Let's demonstrate logging in and out a few times. Note that your `:uid`
;; changes every time you login again with a new anonymous user.
(login-as-anonymous)
(println "(2) user-atm contains ->" @user-atm)

(login-as-anonymous)
(println "(3) user-atm contains ->" @user-atm)
```

```
Usage:
(get-user-atom)

Returns a atom containing either a user-map or nil. Atom will contain nil when
no user is logged into Firestore. Atom will contain a user-map if a user is
currently logged in. user-map has the following form:

{:uid <application_unique_id>
 :email <user_email_address>
 :name <user_identifier>
 :photo <url_to_a_photo_for_this_user>}

Note: :uid will always be present. :email, :name, :photo may be present depending
on sign-in provider and/or whether you have set their values.
```

```
Usage:
(login-as-anonymous)

Log out any existing user, then log in a new anonymous user.
```

You have been logged in as a anonymous user. Anonymous does NOT mean
unidentified (you have a unique user id in `:uid`). Anonymous does however mean that
we don't know your `:email`, `:name`, or `:photo`. Anonymous means that if you
were to logout from this account or loose access to this system, there would be
no way for you to log back in as this anonymous user (though you could always
login as a new anonymous user).

```clojure
;; Of course, you can also logout, let's demonstrate this.
(println "(1) user-atm -> " @user-atm)
(logout)
(println "(2) user-atm ->" @user-atm)
```

```
Usage:
(logout)

Log out the currently logged in user (if any).
```

```clojure
;; Most applications will also need to allow users to delete their accounts.
;; This is trivial in Firestore.

(login-as-anonymous)
(println "Check that we have a user ->" @user-atm)

(delete-user)
(println "We have been logged out as our user is deleted ->" @user-atm)
```

```
Usage:
(delete-user)

Deletes the user specified by user-id from Firestore. This removes all sign-in
providers for this user, as well as deleting the data in the user information
map returned by (get-user-atom). Note that this does NOT delete information
relating to the user from the actual Firestore database.
```

## <a id="clojure_interop"></a> Clojure Interop

```
Usage:
(fire->clj js-object)
(fire->clj js-object opts)

Returns the clojure form of the js-object document from Firestore. opts
allows you to modify the conversion.
```

```
(clj->fire document)
(clj->fire document options)

Returns a javascript object that can be passed to Firestore for writes. opts
allows you to modify the conversion.
```

# <a id="api"></a>API

# <a id="contributing"></a>Contributing

Pull Request are always welcome and appreciated. If you want to discuss firemore, I am available most readily:
1. On [clojurians.slack.com under #firemore](https://clojurians.slack.com/messages/C073DKH9P/).
1. Through the [issue tracking system](https://github.com/samedhi/firemore/issues).
1. By email at stephen@read-line.com .

# <a id="credits"></a>Credits

[Stephen Cagle](https://samedhi.github.io/) is a Senior Software Engineer at [Dividend Finance](https://www.dividendfinance.com/) in San Francisco, CA. He is the original (currently only, but always accepting PRs!) creator/maintainer of firemore.
[@github](https://github.com/samedhi)
[@linkedin](https://www.linkedin.com/in/stephen-cagle-92b895102/)

![Man (Stephen Cagle) holding beer & small dog (Chihuahua)](asset/img/stephen_and_nugget.jpg)

# <a id="License"></a>License

MIT License

Copyright (c) 2019 Stephen Cagle
