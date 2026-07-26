##### Example:

``` java
@RestController  
@RequestMapping("/api")
public class MyRestController {  
  
    @GetMapping("/hello")  
    public String hello() {  
        return "Hello World!";  
    }  
}
```
##### @RestController Annotation:
this annotation composed of other two annotations which are
```java
@Controller  
@ResponseBody  
public @interface RestController{
}
```

`@ResponseBody` = “return data directly to the HTTP response instead of rendering a view.”
“Don’t render a view (like JSP/HTML). Instead, take the return value of this method and write it directly into the HTTP response body.”

##### @GetMapping Annotation:
this annotation composed of other annotation which is
```java
@RequestMapping(  
    method = {RequestMethod.GET}  
)  
public @interface GetMapping {
}
```

---
### Java JSON Data Binding

Data binding is the process of **converting** **JSON data to a Java POJO** (Plain Old Java Object.), also known as Mapping Serialization / Deserialization Marshalling / Unmarshalling

==Spring uses the Jackson Project behind the scenes, Jackson handles data binding between JSON and Java POJO (This done automatically in spring)==

![[Pasted image 20260504234735.png|615]]

---
### JsonMapper Class
`JsonMapper` is a Jackson class used to convert, merge, and update Java objects using JSON data.  
It is especially useful in `PATCH` requests where only specific fields should be updated instead of replacing the entire object.
##### Main Purpose
`JsonMapper` maps JSON fields to Java object fields automatically.
It can:
- Convert JSON → Java objects
- Convert Java objects → JSON
- Partially update existing objects
- Merge request data into entities
##### Example on PATCH HTTP Request
```java
@PatchMapping("/employees/{id}")  
public Employee partialUpdate(@PathVariable int id,  
                              @RequestBody Map<String, String> request) {  
    Employee emp = employeeService.findById(id);  
  
    if (request.containsKey("id")) {  
        throw new RuntimeException("Couldn't update the ID field");  
    }  
  
    Employee patchedEmp = jsonMapper.updateValue(emp, request);  
    return employeeService.update(patchedEmp);  
}
```

* `@RequestBody` is used to read the HTTP request body and convert the incoming JSON data into a Java object automatically, in the above example spring takes the JSON sent by the client and converts it into a `Map<String, String>`.

> [!NOTE] Note
> Without `JsonMapper`, every field must be updated manually, it automates this process and reduces boilerplate code.






