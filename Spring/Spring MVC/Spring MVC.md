## Table of Contents

- [[#1. What is Spring MVC?]]
- [[#2. The Request Life Cycle (DispatcherServlet Flow)]]
- [[#3. Core Components Explained]]
- [[#4. @Controller vs @RestController]]
- [[#5. Request Mapping Annotations]]
- [[#6. Extracting Request Data]]
- [[#7. Building Responses]]
- [[#8. Model, ModelAndView & View Resolution]]
- [[#9. Data Binding & Validation]]
- [[#10. Exception Handling]]
- [[#11. Interceptors (HandlerInterceptor)]]
- [[#12. Content Negotiation]]
- [[#13. File Upload & Download]]
- [[#14. CORS]]
- [[#15. Customizing Spring MVC (WebMvcConfigurer)]]
- [[#16. Testing Spring MVC (MockMvc)]]
- [[#17. Quick Revision Cheat Sheet]]

---

# 1. What is Spring MVC?

**Spring MVC** is Spring's web framework implementing the **Model-View-Controller** pattern on top of the Servlet API. A single front controller (`DispatcherServlet`) receives every HTTP request and delegates work to a chain of pluggable, interface-based components — this is why almost every piece of Spring MVC (handler mapping, view resolution, argument binding) can be swapped or extended.

```
         ┌───────────────────┐
Request →│ DispatcherServlet │→ Response
         └───────────────────┘
                 │
     delegates to pluggable components:
     HandlerMapping, HandlerAdapter,
     ViewResolver, HandlerExceptionResolver, ...
```

> [!info] Servlet stack vs Reactive stack Spring MVC is the **servlet-based**, thread-per-request stack. Its reactive counterpart is **Spring WebFlux** (non-blocking, `Mono`/`Flux`, runs on Netty by default)

---

# 2. The Request Life Cycle (DispatcherServlet Flow)


![[Pasted image 20260531225915.png]]
### Step-by-Step Breakdown

|Step|What Happens|
|---|---|
|**① Request**|A client HTTP request arrives at the server.|
|**② Dispatch to `HandlerMapping`**|`DispatcherServlet` (the **front controller**) asks `HandlerMapping` to find which controller/method should handle this URL.|
|**③ Dispatch to `HandlerAdapter`**|`DispatcherServlet` doesn't invoke the controller directly — it hands off to a `HandlerAdapter`, which knows **how** to actually call that particular type of handler (annotated `@Controller` methods, in modern Spring, are handled by `RequestMappingHandlerAdapter`).|
|**④ Execute the Controller**|The adapter invokes the controller method. The controller delegates to the **Service** layer (business logic), which in turn talks to the **Repository** (data access) and the **Database**.|
|**⑤ Return logical view name**|The controller returns a **logical view name** (a `String`, or embedded in a `ModelAndView`) — _not_ a file path. E.g. `"orderDetails"`, not `/WEB-INF/views/orderDetails.jsp`.|
|**⑥ Set processing result into Model**|Data the controller wants rendered (e.g. an `Order` object) is placed into the **Model** — a `Map`-like data holder passed along with the view name.|
|**⑦ Resolve the View**|`DispatcherServlet` passes the logical view name to a `ViewResolver`, which maps it to an actual renderable `View` (e.g. `orderDetails` → `/WEB-INF/views/orderDetails.jsp`, or a Thymeleaf template).|
|**⑧ Render & Respond**|The resolved `View` renders itself using the Model's data (fills in the template), producing HTML/JSON/etc., which becomes the HTTP response sent back to the client.|

#### For REST Controllers (`@RestController`), Steps ⑤–⑧ Are Skipped

When a handler method (or its class) is annotated with `@ResponseBody` (which `@RestController` implies), there's no view name, no Model, no ViewResolver — the returned object is serialized directly (typically to JSON via Jackson) and written straight to the response body via an `HttpMessageConverter`. 

---

# 3. Core Components Explained

### `DispatcherServlet` — The Front Controller

- A single servlet registered against a URL pattern (commonly `/`) that intercepts **every** incoming request.
- Auto-configured by Spring Boot when `spring-boot-starter-web` is on the classpath — you never register it manually.
- Delegates to a pipeline of strategy interfaces rather than doing any real work itself — this is the **Front Controller** design pattern.

### `HandlerMapping` — "Which controller handles this URL?"

- Maps an incoming request to a **handler** (typically a controller method) based on URL, HTTP method, headers, params, etc.
- The dominant implementation for annotation-based controllers is `RequestMappingHandlerMapping`, which reads `@RequestMapping`/`@GetMapping`/etc. metadata to build the mapping table at startup.
- You can inspect the full mapping table live via Actuator's `/actuator/mappings`.

### `HandlerAdapter` — "How do I actually invoke this handler?"

- Different handler _types_ (annotated methods, `HttpRequestHandler`, legacy `Controller` interface implementations) are invoked differently. The adapter abstracts that away so `DispatcherServlet` doesn't need to know the details.
- For `@Controller`/`@RestController` methods, this is `RequestMappingHandlerAdapter` — it also handles resolving method **arguments** (`@PathVariable`, `@RequestParam`, `@RequestBody`, ...) and processing the **return value**.

### Controller

- A `@Controller`- or `@RestController`-annotated class containing handler methods, one per supported endpoint/action.
- Should stay thin — delegate business logic to the **Service** layer.

### `ViewResolver` — "Which actual template does this logical name point to?"

- Translates a **logical view name** (String) returned by a controller into an actual `View` object capable of rendering.
- Common implementations:

|ViewResolver|Resolves To|
|---|---|
|`InternalResourceViewResolver`|JSPs under a configured prefix/suffix, e.g. `/WEB-INF/views/` + name + `.jsp`|
|`ThymeleafViewResolver`|Thymeleaf HTML templates|
|`FreeMarkerViewResolver`|FreeMarker templates|
|`ContentNegotiatingViewResolver`|Delegates to other resolvers based on the requested content type (JSON vs HTML)|

### `View`

- The object that actually **renders** — takes the Model data and produces the final response body (HTML page, PDF, Excel export, etc.).

### `Model` / `ModelAndView`

- `Model` — a simple attribute holder (`Map<String, Object>`) the controller populates for the view to consume.
- `ModelAndView` — a single object bundling both the model data **and** the view name/`View`, useful when you want to return both explicitly from a method.

---

# 4. `@Controller` vs `@RestController`

||`@Controller`|`@RestController`|
|---|---|---|
|Meta-annotation|`@Component`|`@Controller` + `@ResponseBody`|
|Return value meaning|Logical **view name** (goes through `ViewResolver`)|The actual **response body** (serialized directly)|
|Typical use|Server-rendered HTML pages (Thymeleaf, JSP)|REST APIs (JSON/XML)|
|Needs `@ResponseBody` per method?|Yes, if you want a raw body from a specific method in an otherwise view-returning controller|No — implied on every method|

```java
@Controller
public class PageController {

    @GetMapping("/orders/{id}")
    public String orderDetails(@PathVariable Long id, Model model) {
        model.addAttribute("order", orderService.findById(id));
        return "orderDetails"; // logical view name → resolved by ViewResolver
    }
}
```

```java
@RestController
@RequestMapping("/api/orders")
public class OrderApiController {

    @GetMapping("/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        return orderService.findById(id); // serialized straight to JSON
    }
}
```

### How Serialization Happens: `HttpMessageConverter`

- With `@ResponseBody`, Spring picks an `HttpMessageConverter` based on the request's `Accept` header (and the object type) to convert the return value into bytes.
- `MappingJackson2HttpMessageConverter` (JSON, via Jackson) is registered by default when Jackson is on the classpath — which `spring-boot-starter-web` includes automatically.

---

# 5. Request Mapping Annotations

### `@RequestMapping` — The General-Purpose Form

```java
@RequestMapping(value = "/orders", method = RequestMethod.GET)
public List<OrderDto> listOrders() { ... }
```

### HTTP-Method-Specific Shortcuts (Preferred)

|Annotation|Equivalent To|
|---|---|
|`@GetMapping`|`@RequestMapping(method = RequestMethod.GET)`|
|`@PostMapping`|`@RequestMapping(method = RequestMethod.POST)`|
|`@PutMapping`|`@RequestMapping(method = RequestMethod.PUT)`|
|`@DeleteMapping`|`@RequestMapping(method = RequestMethod.DELETE)`|
|`@PatchMapping`|`@RequestMapping(method = RequestMethod.PATCH)`|

### Class-Level + Method-Level Composition

```java
@RestController
@RequestMapping("/api/orders")     // shared prefix for all methods below
public class OrderApiController {

    @GetMapping                    // GET /api/orders
    public List<OrderDto> listAll() { ... }

    @GetMapping("/{id}")           // GET /api/orders/{id}
    public OrderDto getOne(@PathVariable Long id) { ... }

    @PostMapping                   // POST /api/orders
    public OrderDto create(@RequestBody OrderDto dto) { ... }
}
```

### Narrowing a Mapping Further

```java
@GetMapping(
    path = "/orders",
    params = "status=active",          // only matches ?status=active
    headers = "X-Api-Version=2",       // only matches with this header
    produces = "application/json",     // only matches if client accepts JSON
    consumes = "application/json"      // only matches if request body is JSON
)
```

---

# 6. Extracting Request Data

|Annotation|Extracts From|Example|
|---|---|---|
|`@PathVariable`|URI template segment|`@GetMapping("/orders/{id}")` + `@PathVariable Long id`|
|`@RequestParam`|Query string / form param|`?status=active` + `@RequestParam String status`|
|`@RequestBody`|Entire request body (deserialized via `HttpMessageConverter`)|`@RequestBody OrderDto dto`|
|`@RequestHeader`|A specific HTTP header|`@RequestHeader("Authorization") String token`|
|`@CookieValue`|A specific cookie|`@CookieValue("sessionId") String sessionId`|
|`@MatrixVariable`|Matrix-style URI params (`/orders;color=red`)|Rare, needs explicit enabling|
|`@ModelAttribute`|Binds request params to a Java object's fields (form binding)|`@ModelAttribute OrderForm form`|

### Examples

```java
@GetMapping("/orders/{id}")
public OrderDto getOrder(@PathVariable Long id) { ... }

@GetMapping("/orders")
public List<OrderDto> search(
        @RequestParam(required = false, defaultValue = "all") String status,
        @RequestParam(name = "page", defaultValue = "0") int page) { ... }

@PostMapping("/orders")
public OrderDto create(@RequestBody OrderDto dto) { ... }

@GetMapping("/secure")
public String secure(@RequestHeader("Authorization") String token) { ... }
```

> [!tip] `@RequestParam` vs `@ModelAttribute` for forms `@RequestParam` pulls individual values one field at a time. `@ModelAttribute` binds a **whole object's** fields from matching request parameter names in one shot — much cleaner for HTML forms with many fields.

---

# 7. Building Responses

### Plain Object (with `@ResponseBody` / `@RestController`)

```java
@GetMapping("/orders/{id}")
public OrderDto getOrder(@PathVariable Long id) {
    return orderService.findById(id); // 200 OK, body = serialized OrderDto
}
```

### `ResponseEntity<T>` — Full Control Over Status, Headers, and Body

```java
@PostMapping("/orders")
public ResponseEntity<OrderDto> create(@RequestBody OrderDto dto) {
    OrderDto saved = orderService.save(dto);
    URI location = URI.create("/api/orders/" + saved.getId());
    return ResponseEntity.created(location).body(saved); // 201 Created + Location header
}

@GetMapping("/orders/{id}")
public ResponseEntity<OrderDto> getOrder(@PathVariable Long id) {
    return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElseGet(() -> ResponseEntity.notFound().build()); // 404 if absent
}
```

### `@ResponseStatus` — Fixed Status Code

```java
@ResponseStatus(HttpStatus.CREATED)
@PostMapping("/orders")
public OrderDto create(@RequestBody OrderDto dto) { ... }
```

Less flexible than `ResponseEntity` (can't vary the status conditionally), but concise for a single fixed outcome.

### Redirect vs Forward (View-Returning Controllers)

```java
@PostMapping("/orders")
public String create(OrderForm form) {
    orderService.save(form);
    return "redirect:/orders";   // new HTTP redirect — browser URL changes
}

@GetMapping("/legacy-orders")
public String legacy() {
    return "forward:/orders";    // internal forward — same request, no new HTTP round-trip
}
```

---

# 8. Model, `ModelAndView` & View Resolution

### Adding Data for the View

```java
@GetMapping("/orders/{id}")
public String orderDetails(@PathVariable Long id, Model model) {
    model.addAttribute("order", orderService.findById(id));
    return "orderDetails";
}
```

### Using `ModelAndView` Directly

```java
@GetMapping("/orders/{id}")
public ModelAndView orderDetails(@PathVariable Long id) {
    ModelAndView mav = new ModelAndView("orderDetails");
    mav.addObject("order", orderService.findById(id));
    return mav;
}
```

### View Resolution Chain

Multiple `ViewResolver`s can be registered; Spring tries them **in order** (controlled by `@Order`/`order` property) until one returns a non-null `View`. `ContentNegotiatingViewResolver` is often placed first to pick JSON vs HTML based on the `Accept` header, falling through to a template-engine resolver otherwise.

---

# 9. Data Binding & Validation

### Bean Validation with `@Valid`

```java
public class OrderForm {
    @NotBlank
    private String customerName;

    @Min(1)
    private int quantity;

    @Email
    private String contactEmail;
    // getters/setters
}
```

```java
@PostMapping("/orders")
public ResponseEntity<?> create(@Valid @RequestBody OrderForm form, BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        return ResponseEntity.badRequest().body(bindingResult.getAllErrors());
    }
    orderService.save(form);
    return ResponseEntity.ok().build();
}
```

> [!warning] `BindingResult` must come **immediately** after the `@Valid`/`@Validated` parameter If you omit `BindingResult`, validation failures throw `MethodArgumentNotValidException` instead — often actually preferable, since you can handle it centrally (see [[#10. Exception Handling]]) rather than checking `hasErrors()` in every method.

### Common Validation Annotations

|Annotation|Rule|
|---|---|
|`@NotNull`|Value must not be `null`|
|`@NotBlank`|String must not be `null`/empty/whitespace-only|
|`@NotEmpty`|Collection/String must not be `null`/empty (whitespace allowed for Strings)|
|`@Size(min=, max=)`|Length/size bounds|
|`@Min` / `@Max`|Numeric bounds|
|`@Email`|Valid email format|
|`@Pattern(regexp=)`|Must match a regex|
|`@Past` / `@Future`|Date must be in the past/future|

### Custom `Converter`/`Formatter`

For binding non-trivial types (e.g. a `String` param → a custom `Enum` or value object), register a `Converter<S, T>` bean or via `WebMvcConfigurer.addFormatters(...)`.

---

# 10. Exception Handling

### `@ExceptionHandler` — Local to One Controller

```java
@RestController
public class OrderApiController {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<String> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

### `@ControllerAdvice` / `@RestControllerAdvice` — Global, Across All Controllers

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return ResponseEntity.badRequest().body(errors);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.internalServerError().body(new ErrorResponse("Unexpected error"));
    }
}
```

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`, mirroring the `@RestController` relationship.

### Resolution Order

Spring picks the **most specific matching handler**: method-local `@ExceptionHandler` in the same controller wins over one declared in a `@ControllerAdvice`; within `@ControllerAdvice`, the most specific exception type match wins over a broader one (e.g. `OrderNotFoundException` handler beats a catch-all `Exception` handler).

### `@ResponseStatus` on a Custom Exception

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String message) { super(message); }
}
```

Simple alternative when you don't need a custom response body — just throw it, and Spring maps it to the given status automatically.

---

# 11. Interceptors (`HandlerInterceptor`)

Interceptors sit **between** `DispatcherServlet` and the controller — closer to the request-handling pipeline than AOP, but higher-level than a raw servlet `Filter`. 

```java
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        System.out.println("Before handler: " + request.getRequestURI());
        return true; // false = short-circuit, controller never runs
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                            Object handler, ModelAndView modelAndView) {
        System.out.println("After handler, before view rendering");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 Object handler, Exception ex) {
        System.out.println("After the complete request has finished (view rendered)");
    }
}
```

### Registering an Interceptor

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

### Interceptor vs Filter vs AOP — Where Each Fits

```
HTTP Request
      │
  Servlet Filter     ← web-container level, before Spring even sees the request
      │
DispatcherServlet
      │
HandlerInterceptor   ← Spring MVC level, knows about the handler/controller
      │
   Controller
      │
   Service
      │
  AOP Proxy    ← Spring bean/method level, works on any bean not just controllers
      │
Target Method
```

||Filter|Interceptor|AOP|
|---|---|---|---|
|Layer|Servlet container|Spring MVC|Any Spring bean|
|Knows about the matched controller/handler?|No|Yes|No (works on method signatures)|
|Can access `ModelAndView`?|No|Yes (`postHandle`)|No|
|Typical use|Auth, CORS, encoding|Logging, auth per-route, i18n locale setup|Transactions, business-method logging|

---

# 12. Content Negotiation

Spring MVC can serve different representations (JSON vs XML vs HTML) of the same resource based on what the client asks for.

### Via `Accept` Header (Default Strategy)

```
Accept: application/json   → JSON response
Accept: application/xml    → XML response (if a converter is registered)
```

### Via URL Suffix / Parameter (Legacy, off by default)

```
GET /orders/1.json
GET /orders/1?format=json
```

### `produces` / `consumes` on Mappings

```java
@GetMapping(value = "/orders/{id}", produces = "application/json")
@PostMapping(value = "/orders", consumes = "application/json")
```

---

# 13. File Upload & Download

### Upload

```java
@PostMapping("/upload")
public ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) throws IOException {
    Files.write(Paths.get("/uploads/" + file.getOriginalFilename()), file.getBytes());
    return ResponseEntity.ok("Uploaded: " + file.getOriginalFilename());
}
```

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

### Download

```java
@GetMapping("/files/{filename}")
public ResponseEntity<Resource> download(@PathVariable String filename) throws IOException {
    Resource file = new FileSystemResource("/uploads/" + filename);
    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
            .body(file);
}
```

---

# 14. CORS

### Per-Controller/Method

```java
@CrossOrigin(origins = "https://myfrontend.com")
@RestController
public class OrderApiController { ... }
```

### Global Configuration

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://myfrontend.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true);
    }
}
```

---

# 15. Customizing Spring MVC (`WebMvcConfigurer`)

A single interface for hooking into many MVC customization points without writing raw `@Configuration`-heavy XML-era setup:

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home"); // no controller class needed
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
    }

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver()); // custom @CurrentUser injection
    }
}
```

> [!warning] `@EnableWebMvc` — usually NOT needed in Spring Boot `@EnableWebMvc` turns on **manual** Spring MVC configuration, which **disables** Spring Boot's MVC auto-configuration defaults. In a Boot app, just implement `WebMvcConfigurer` to _add to_ the auto-configured setup — don't add `@EnableWebMvc` unless you deliberately want full manual control.

---

# 16. Testing Spring MVC (`MockMvc`)


```java
@WebMvcTest(OrderApiController.class)
class OrderApiControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private OrderService orderService;

    @Test
    void shouldReturnOrder() throws Exception {
        when(orderService.findById(1L)).thenReturn(Optional.of(new OrderDto(1L, "Widget")));

        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.id").value(1))
               .andExpect(jsonPath("$.name").value("Widget"));
    }

    @Test
    void shouldReturn404WhenMissing() throws Exception {
        when(orderService.findById(99L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/orders/99"))
               .andExpect(status().isNotFound());
    }
}
```

`@WebMvcTest` loads **only** the web layer (controllers, `@ControllerAdvice`, converters) — service/repository beans must be mocked with `@MockBean`, which keeps these tests fast and focused.

---

# 17. Quick Revision Cheat Sheet

|Concept|Remember|
|---|---|
|`DispatcherServlet`|Front controller — receives every request|
|`HandlerMapping`|Finds _which_ controller/method handles the URL|
|`HandlerAdapter`|Knows _how_ to invoke that handler type|
|Controller return value|A **logical view name**, not a file path (unless `@ResponseBody`/`@RestController`)|
|`ViewResolver`|Logical name → actual `View`|
|`Model`|Data passed from controller to the view|
|`@Controller`|Returns view names, rendered server-side|
|`@RestController`|`@Controller` + `@ResponseBody` — returns data serialized directly|
|`@PathVariable`|From the URL path|
|`@RequestParam`|From query string / form field|
|`@RequestBody`|From the whole request body|
|`ResponseEntity`|Full control: status + headers + body|
|`@ExceptionHandler`|Local exception handling|
|`@ControllerAdvice`|Global exception handling / cross-cutting controller logic|
|`HandlerInterceptor`|MVC-level request interception (pre/post/afterCompletion)|
|Filter vs Interceptor vs AOP|Servlet-level → MVC-level → any-bean-level|
|`@EnableWebMvc`|Skip it in Spring Boot — it disables auto-configuration|
|`MockMvc` + `@WebMvcTest`|Fast, web-layer-only controller tests|

---



