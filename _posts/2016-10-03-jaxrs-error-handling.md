---
layout: post
title:  JAX-RS - Error Handling
---

Error handling strategy for JAX-RS Runtime Exceptions

Even though we can use any class that extends `WebApplicationException` from jax rs packagge, there is a possibility to create custom exception mappers.

#### Exception Mapper for Runtime Exception

Just add this class to your project and you will be rocking a JSON/XML proper response.

```java
import static javax.ws.rs.core.Response.Status.INTERNAL_SERVER_ERROR;

@Provider
@Produces({"application/json", "application/xml"})
public class RuntimeExceptionMapper implements ExceptionMapper<RuntimeException> {

    @Override
    public Response toResponse(RuntimeException e) {
        return Response.status(INTERNAL_SERVER_ERROR)
                .entity(new Error(e.getMessage()))
                .build();
    }

    @XmlRootElement
    @XmlAccessorType(XmlAccessType.FIELD)
    private static class Error implements Serializable {

        String msg;

        public Error() {
        }

        public Error(String name) {
            this.msg = name;
        }
    }
}
```

After this we can safely throw Runtime Exceptions without exposing internal details to the client.

```java
  @GET
    @Path("hello/{param}")
    public Response hello(@PathParam("param") String param) {
        throw new IllegalArgumentException("invalid param");
    }
```

No dependencies needed.
This will produce the following Json:

```json
{
	"msg": "invalid param"
}
```

****


