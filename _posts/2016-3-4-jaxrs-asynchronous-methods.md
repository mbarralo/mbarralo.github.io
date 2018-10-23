---
layout: post
title: Rest Asynchronous Server
tags: javaee rest
icon: images/java.png
---

How to achieve maximum throughput with asynchronous JAX-RS.

How to allow http-threads to serve the requests without effort, as well as combining asynchronous results from different remote services: 

(Even with the default 5 threads of `Payara` some amazing results were delivered.)

***

#### Asynchronous JAX-RS with Service Orchestration - :)

```java
@Path("/async")
@ApplicationScoped
public class AsyncResource {

    @Resource
    private ManagedExecutorService mes;

    @GET
    public void compute(@Suspended final AsyncResponse asyncResponse) {

        asyncResponse.setTimeout(2000, TimeUnit.MILLISECONDS);
        asyncResponse.setTimeoutHandler(resp -> resp.resume(Response.status(REQUEST_TIMEOUT).build()));

        mes.execute(() -> {
            try {
                CompletableFuture<Integer> future = supplyAsync(() -> first(), mes)
                        .thenCombine(supplyAsync(() -> second(), mes), (a, b) -> a + b);

                asyncResponse.resume(future.get());
            } catch (Exception e) {
                asyncResponse.resume(Response.serverError().build());
            }
        });
    }

    private Integer first() {
        sleep(1000);
        return 1;
    }

    private Integer second() {
        sleep(1000);
        return 1;
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

****

