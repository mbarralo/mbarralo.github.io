---
layout: post
title:  JAX-RS - Asynchronous Client
---

Simple JAX-RS client for asynchronous calls.

This follows up on the previous blog post [Asynchronous Methods](/jaxrs-asynchronous-methods).

#### Maven Dependency

Requires only a simple maven dependency.

```xml
<dependency>
    <groupId>org.glassfish.jersey.core</groupId>
    <artifactId>jersey-client</artifactId>
    <version>2.22</version>
</dependency>

<!-- optional for json marshalling -->
<dependency>
    <groupId>org.glassfish.jersey.media</groupId>
    <artifactId>jersey-media-json-jackson</artifactId>
    <version>2.22</version>
</dependency>
```

#### Simple Example

Here is the code for the client. This is a simple example.


```java
public static void main(String[] args) throws InterruptedException, ExecutionException {
    long start = System.nanoTime();

    final Client client = ClientBuilder.newClient();
    final AsyncInvoker asyncInvoker = client.target(BASE_URI).path("async").request().async();

    Future<Integer> a = asyncInvoker.get(Integer.class);
    Future<Integer> b = asyncInvoker.get(Integer.class);
    System.out.println(a.get()+b.get());

    System.out.println("check -> finished in " + (System.nanoTime() - start) / 1_000_000 + " ms");
    client.close();
    System.exit(0);
}
```


#### Complex Example


A more complex example.. Say for instance that we want to coordinate several asynchronous requests.
We can use a utility class `CountDownLatch`


```java
public static void main(String[] args){

    long start = System.nanoTime();

    final Queue<String> errors = new ConcurrentLinkedQueue<String>();
    final CountDownLatch latch = new CountDownLatch(1);

    final Client client = ClientBuilder.newClient();

    Future<Response> future = client.target(URI).request().async().get(new InvocationCallback<Response>() {
        @Override
        public void completed(Response response) {
            System.out.println(String.format("Response {status: %d, body: %s}", response.getStatus(), response.readEntity(String.class)));
            latch.countDown();
        }

        @Override
        public void failed(Throwable error) {
            errors.offer(String.format("Request has failed: %s", error.toString()));
            latch.countDown();
        }
    });

    try {
        if (!latch.await(3, TimeUnit.SECONDS)) {
            errors.offer("Waiting for requests to complete has timed out.");
        }
    } catch (InterruptedException e) {
        errors.offer("Waiting for requests to complete has been interrupted.");
    }

    if (!errors.isEmpty()) {
        printErrors(errors);
    }

    System.out.println("finished in " + (System.nanoTime() -start)/ 1_000_000 + " ms.");

    client.close();
    System.exit(errors.size() > 0 ? -1 : 0);
}

private static void printErrors(Queue<String> errors) {
    System.out.println("Following errors occurred during the request execution");
    for (String error : errors) {
        System.out.println("\t" + error);
    }
}
```

Small examples to use the asynch syntax of (jersey) jax-rs client.


****