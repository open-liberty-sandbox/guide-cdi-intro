# feature/my-solution — Change Log

This document summarises the two commits added on top of `prod` in the `feature/my-solution` branch.

---

## 1. Make `InventoryManager` an application-scoped CDI bean (`InventoryManager.java`)

**Commit:** `feat: make InventoryManager an application-scoped CDI bean`

### What changed

`InventoryManager.java` was updated to become a managed CDI bean by adding the `@ApplicationScoped` annotation:

```java
@ApplicationScoped
public class InventoryManager {
    private List<SystemData> systems = Collections.synchronizedList(new ArrayList<>());

    public void add(String hostname, Properties systemProps) { ... }

    public InventoryList list() { ... }
}
```

### Why

Without a CDI scope annotation the class is just a plain Java object and cannot be injected or managed by the container. `@ApplicationScoped` creates a single shared instance for the lifetime of the application, which is the correct behaviour for an in-memory inventory that must accumulate entries across many HTTP requests. A narrower scope (e.g., `@RequestScoped`) would create a fresh, empty instance per request and lose all previously recorded hosts.

---

## 2. Inject dependencies into `InventoryResource` (`InventoryResource.java`)

**Commit:** `feat: inject InventoryManager and SystemClient into InventoryResource`

### What changed

`InventoryResource.java` was updated to remove manual object construction and instead receive its dependencies through CDI field injection:

```java
@ApplicationScoped
@Path("/systems")
public class InventoryResource {

    @Inject
    InventoryManager manager;

    @Inject
    SystemClient systemClient;

    @GET
    @Path("/{hostname}")
    @Produces(MediaType.APPLICATION_JSON)
    public Response getPropertiesForHost(@PathParam("hostname") String hostname) {
        Properties props = systemClient.getProperties(hostname);
        if (props == null) {
            return Response.status(Response.Status.NOT_FOUND)
                    .entity("{ \"error\" : \"Unknown hostname " + hostname + "...\" }")
                    .build();
        }
        manager.add(hostname, props);
        return Response.ok(props).build();
    }

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public InventoryList listContents() {
        return manager.list();
    }
}
```

### Why

Using `@Inject` hands lifecycle and wiring responsibility to the CDI container. `InventoryResource` itself is annotated `@ApplicationScoped` so that the single `InventoryManager` instance (from commit 1) is shared across all requests — the container injects the same bean into every injection point of the same scope. `SystemClient` is also injected rather than instantiated directly, keeping `InventoryResource` decoupled from its collaborators and making it easier to substitute alternative implementations in tests.
