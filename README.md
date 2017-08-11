# xml-in

your friendly XML navigator

[![Clojars Project](http://clojars.org/tolitius/xml-in/latest-version.svg)](http://clojars.org/tolitius/xml-in)

## What

XML is this new hot markup language everyone is raving about. Attributes, namespaces, schemas, security, XSL.. what's there not to love.

`xml-in` is not about parsing XML, but rather working with already parsed XML.
It takes heavily nested `{:tag .. :attrs .. :content [...]}` structures that Clojure XML parsers produce and allows to navigate
this structures in a Clojure's "`get-in` style" using internal or custom transducers.

[clojure/data.xml](https://github.com/clojure/data.xml) is an example of a good and lazy Clojure/ClojureScript XML parser
[funcool/tubax](https://github.com/funcool/tubax) is another example of ClojureScript XML parser

## Why

The most common XML navigation is done via zippers. [clojure/data.zip](https://github.com/clojure/data.zip) is usially used
and a common navigation looks like this:

```clojure
(data.zip/xml1-> (clojure.zip/xml-zip parsed-xml)
                 :universe
                 :system
                 :delta-orionis
                 :δ-ori-aa1
                 :radius
                 data.zip/text)
```

There is a great article [XML for fun and profit](http://blog.korny.info/2014/03/08/xml-for-fun-and-profit.html) that shows how zippers
are used to navigate XML DOM trees.

But we can do better: faster, cleaner, composable and no zippers.

How much faster? Let's see:

zippers
```clojure
boot.user=> (time (dotimes [_ 250000] (data.zip/xml1-> (clojure.zip/xml-zip parsed-xml) :universe :system :delta-orionis :δ-ori-aa1 :radius data.zip/text)))
"Elapsed time: 13385.563442 msecs"
```

xml-iz
```clojure
boot.user=> (time (dotimes [_ 250000] (xml/find-first parsed-xml [:universe :system :delta-orionis :δ-ori-aa1 :radius])))
"Elapsed time: 885.660409 msecs"
```

## Most common navigation

Here is an XML document all the examples are based on:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<universe>
  <system>
    <solar>
      <planet age="4.543" inhabitable="true">Earth</planet>
      <planet age="4.503">Mars</planet>
    </solar>
    <delta-orionis>
      <constellation>Orion</constellation>
      <δ-ori-aa1>
        <mass>24</mass>
        <radius>16.5</radius>
        <luminosity>190000</luminosity>
        <surface-gravity>3.37</surface-gravity>
        <temperature>29500</temperature>
        <rotational-velocity>130</rotational-velocity>
      </δ-ori-aa1>
    </delta-orionis>
  </system>
</universe>
```

it lives in [dev-resources/universe.xml](dev-resources/universe.xml)

Since `xml-in` works with a parsed XML (e.g. a DOM tree), let's parse it once and call it `universe`:

```clojure
=> (require '[clojure.data.xml :as dx])
=> (def universe (dx/parse-str (slurp "dev-resources/universe.xml")))
#'boot.user/universe
```

it gets parsed in a common nested `{:tag :attrs :content}` structure that looks like this:

```clojure
=> (pprint universe)
{:tag :universe,
 :attrs {},
 :content
 ("\n  "
  {:tag :system,
   :attrs {},
   :content
   ("\n    "
    {:tag :solar,
     :attrs {},
     :content
     ("\n      "
      {:tag :planet,
      ;; ...
      ;; ...
```

In order to access child nodes `xml-in` takes a direct vector path to them. For example, let's check out "those two" planets in a solar system.

Bringint `xml-in` in:

```clojure
=> (require '[xml-in.core :as xml])
```

and

```clojure
=> (xml/find-all universe [:universe :system :solar :planet])
("Earth" "Mars")
```

All the planets are returned. In case we need "a" planet we can match the first one and stop searching:

```clojure
=> (xml/find-first universe [:universe :system :solar :planet])
"Earth"
```

notice `find-all` vs. `find-first`

### All matching vs. The first matching

Even if there is only one element that matches a search criteria it is best to not look for it using `find-all`
since there is a cost of _looking_ at all the child nodes that are on the same level and a matched element.

Let's look at the example. From the XML above, let's find a `radius` of `δ-ori-aa1` in a `delta-orionis` star system:

```clojure
boot.user=> (xml/find-all universe [:universe :system :delta-orionis :δ-ori-aa1 :radius])
"16.5"
boot.user=> (xml/find-first universe [:universe :system :delta-orionis :δ-ori-aa1 :radius])
"16.5"
```

Both `find-all` and `find-first` return the same exact value, but we know that there `δ-ori-aa1` has only one `radius`.

Let's see the performance difference:

```clojure
=> (time (dotimes [_ 250000] (xml/find-all universe [:universe :system :delta-orionis :δ-ori-aa1 :radius])))
"Elapsed time: 1382.708085 msecs"
```

```clojure
boot.user=> (time (dotimes [_ 250000] (xml/find-first universe [:universe :system :delta-orionis :δ-ori-aa1 :radius])))
"Elapsed time: 808.148405 msecs"
```

Quite a difference. The secret is quite simple: `find-first` stops searching once it finds a matching element.
But it does improve performance, especially for a large number of XML documents.

## Functional navigation

While a common property based `[:a :b :c :d]` navigation 

## Creating sub documents

## License

Copyright © 2017 tolitius

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
