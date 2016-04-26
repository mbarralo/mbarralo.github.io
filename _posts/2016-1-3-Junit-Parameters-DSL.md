---
layout: post
title: Junit - Scenarios Helper Class
---

This is just a small example of a simple Helper class to use declarative parameterized tests with the given-when-then syntax.

This is a simple example of things one can do with simple domain specific languages (**DSL**) and/or **Helper** classes (e.g. emulate the [Specification By Example](http://martinfowler.com/bliki/GivenWhenThen.html)). 

This type of syntax can be found in some very sophisticated frameworks such as [Spock](https://github.com/spockframework) and [Cucumber BDD](https://cucumber.io)

### DSL for Custom inline Junit Parameters

```java
public class StringsTest {

    @Test
    public void stringLength() {

        Scenarios
                .given("john", "doe")
                .when(String::length) // function under test
                .expect(4, 3)
                .build();
    }
}
```

And here is the `Scenarios` class..

```java
class Scenarios<T, R> {

    private List<T> parameters;
    private List<R> results;
    private Function<T, R> function;

    private Scenarios(T... parameters) {
        this.parameters = Arrays.asList(parameters);
    }

    public static <T, R> Scenarios<T, R> given(T... parameters) {
        return new Scenarios<T, R>(parameters);
    }

    public Scenarios<T, R> expect(R... results) {
        this.results = Arrays.asList(results);
        return this;
    }

    public Scenarios<T, R> when(Function<T, R> mapFunction) {
        this.function = mapFunction;
        return this;
    }

    public void build() {
        List<R> collect = parameters.stream().map(function).collect(Collectors.toList());
        assertEquals(results, collect);
    }
}
```

Junit also contains support for Parameters out-of-the-box.

****
----

