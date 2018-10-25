---
layout: post
title: Java 8 Features
tags: java
icon: images/java.png
---

Java 8 was a major release and brought important new features for the Java language.
Here are the highlights:

## 1. Lambdas

Java 8 added support for functional programming through __lambas__. Higher order functions are now possible in Java.

```java

public interface ScolarshipPolicy {
    boolean isEligible(Student student);
}

private boolean isElgibleForScolarship(Student student, ScolarshipPolicy scolarshipPolicy) {
    return scolarshipPolicy.isEligible(student);
}

// before lambdas
boolean isEligible = isElgibleForScolarship(chuck, new ScolarshipPolicy() {
    @Override
    public boolean isEligible(Student student) {
        return student.getScore() > 10.0;
    }
});

//now
isElgibleForScolarship(chuck, (student) -> student.getScore() > 10.0);
isElgibleForScolarship(chuck, (student) -> student.getScore() > 10.0 && !student.hasScolarship());

```

#### Functional Interfaces

- Specifies exactly one abstract method
- Describes the signature of the lambda expression
- Examples already in java 7: Comparable, Runnable and Callable

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t); //Predicate function descriptor
}
```

#### Method References

- Reuse existing method definitions and pass them as lambdas
- Increase readability

```java
//before 
(Student a) -> a.getScore()
sort((s1, s2) -> s1.getScore().compareTo(s2.getScore()))

//now
Student::getScore
sort(comparing(Student::getScore))
```

## 2. Streams


Java 8 added a really nice feature called Streams. This changed the way we program in a much more intuitively and declarative way.

Example:
```java
// before streams
List<Student> filteredStudents = new ArrayList<Student>();
for (Student s : classRoom) {
    if (s.getScore() > 10.0) {
        filteredStudents.add(s);
    }
}

Collections.sort(filteredStudents, new Comparator<Student>() {
    @Override
    public int compare(Student o1, Student o2) {
        return Double.compare(o1.getScore(), o2.getScore());
    }
});

List<String> namesList = new ArrayList<>();
for (Student s : filteredStudents) {
    namesList.add(s.getName());
}


// now
List<String> namesList =
    classRoom.stream()
        .filter(s -> s.getScore() > 10.0)
        .sorted(comparing(Student::getScore))
        .map(Student::getName)
        .collect(toList());
``` 

The code is much more concise, comparison is readable now and we are not using throw-away variables.


## 3. Default Methods

* Java 8 enables interface default methods 
* Interfaces now have behaviour
    * You can implement static methods in Interfaces
    * You can add default methods to interfaces and implementing classes may choose to override.

```java
default void sort(Comparator<? super E> c){
    Collections.sort(this, c);
}
```

## 4. Optional

**Problems with null**
* It’s a source of error. NullPointerException is by far the most common exception in Java.
* It bloats your code. It worsens readability by making it necessary to fill your code with often deeply nested null checks.
* It’s meaningless. It doesn’t have any semantic meaning
* etc

How can we avoid returning null?
```java
// before
public String getCarModel(String licencePlate) {
    Car car = carDao.findCar(licencePlate);
    if (car != null) {
      return car.getModel();
    }
    return null;
}

// now
public Optional<String> getCarModelImproved(String licencePlate) {
    Optional<Car> car = carDao.searchFor(licencePlate);
    return car.map(Car::getModel);
}
```

## 5. Completable Futures

**What are Futures?**

Some computation to be done in the future, asynchronously, allowing the main thread of execution to continue.

```java

ExecutorService executorService = Executors.newCachedThreadPool();
Future<Double> futureValue = executorService.submit(new Callable<Double>() {
    @Override
    public Double call() throws Exception {
        Thread.sleep(1000);
        return 10.0;
    }
});

System.out.println("waiting");
// do something else

try {
    Double result = futureValue.get(); // blocking operation
    System.out.println(result);
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}
```

Problems with Futures:

* Difficult to express dependencies between results of a Future
* Hard to work with. Use of timeouts and isDone methods are not helpful in building a working asynchronous/parallel pipeline.
* No declarative syntax to:
* Combine two asynchronous computations in one
    * Waiting for the completion of all tasks performed by a set of Futures
    * Waiting for the completion of the quickest task in a set of Futures
    * Reacting to a future completion (being notified when the completion happens and do further processing with the result)


#### Asynchronous pipelines

```java
List<CompletableFuture<String>> priceFutures = shops.stream()
    .map(shop -> CompletableFuture.supplyAsync(() -> shop.getPrice("iphone6")))
    .map(future -> future.thenApply(Quote::parse))
    .map(future -> future.thenCompose(quote -> CompletableFuture.supplyAsync(() ->
                Discount.applyDiscount(quote))))
    .collect(toList());

List<String> prices = priceFutures.stream().map(CompletableFuture::join).collect(toList());
```

#### Combining Values

Combining Results from asynchronous computations:
```java
Future<Integer> combine =
    CompletableFuture.supplyAsync(() -> value(2).withDelay(1000))
    .thenCombine(CompletableFuture.supplyAsync(() -> value(3).withDelay(1000)),
        (a, b) -> a + b);
```

The solution before takes 1 second to complete instead of the 2 seconds if using serial processing.

#### Reacting

```java
CompletableFuture[] futures = findPricesStream("iphone6")
    .map(p -> p.thenAccept(System.out::println))
    .toArray(size -> new CompletableFuture[size]);

CompletableFuture.allOf(futures).join();
```

## 6. Date and Time API

Before Java 8 there was no standard for Date and Time API. That led to error prone code:

* Date represents a point in time (Timestamp -> long)
* The years start from 1900 and the months start at index 0
    * 19-04-2016 = new Date(116,3,19)
* Timezone problems
    * Lisbon is WET not BST
* Date and Calendar confusion
* DateFormat only works with Date
* DateFormat not Thread-safe
* Mutability in Date and Calendar leads to maintenance overhead and error-prone code


For those reasons a new Date and Time API was created -> java.time package

Important Classes:
- LocalDate
- LocalTime
- LocalDateTime
- Instant
- Duration
- Period

```java

LocalDate date = LocalDate.of(2016, 4, 20);
int year = date.getYear();
Month month = date.getMonth();
int day = date.getDayOfMonth();
DayOfWeek dayOfWeek = date.getDayOfWeek();
// [2016, APRIL, 20, WEDNESDAY, 30, true]

LocalDate today = LocalDate.now();
//2016-04-20

LocalTime time = LocalTime.of(14, 30, 01);
time.getHour();
time.getMinute();
time.getSecond();
// [14, 30, 01]

```

#### Parsing

```java
LocalDate localDate = LocalDate.parse("2016-04-20");
LocalTime localTime = LocalTime.parse("16:30:01");

DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MM-yyyy");
LocalDate parsedDate = LocalDate.parse("20-04-2016", formatter);

Assert.assertEquals(localDate, parsedDate);

System.out.println(parsedDate.format(formatter));

//20-04-2016
```

#### Immutability

All classes in the new date time API are immutable:

```java
LocalDate date = LocalDate.of(2016, 4, 20);
date = date.plusYears(2).minusDays(10);
date.withYear(2011);

LocalDate date2 = date.plusWeeks(3);
LocalDate date3 = date.withYear(2011);

// date = 2018-04-10, date2 = 2018-05-01, date3 = 2011-04-10
```


****

