`We want to handle the exception and return response with error body and status as JSON object`
##### Development Process
1. Create a custom error response class (Java POJO)
2. Create a custom exception class 
3. Update REST service to throw exception under specific conditions
4. Add an exception handler method using **@ExceptionHandler**
`Jackson will handle converting this error response to JSON`


> [!NOTE] Exception Handler Method
> A method annotated with **@ExceptionHandler** annotation, which **maps** a thrown exception with specific type to a **ResponseEntity** object (provides fine-grained control to HTTP status code, HTTP headers and Response body)

``` java
@ExceptionHandler  
public ResponseEntity<StudentErrorResponse> handleException(StudentNotFoundException ex) {  
    StudentErrorResponse error = new StudentErrorResponse(  
            HttpStatus.NOT_FOUND.value(),  
            ex.getMessage(),  
            System.currentTimeMillis()  
    );  
  
    return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);  
}

---------------------------------------------------------------
@ExceptionHandler({Exception1.class, Exception2.class, ...}) 
public ResponseEntity<StudentErrorResponse> handleException() {  
}
```

* Method signature must be as follows:
	* Arg -> Type of exception that needed to be handled
	* Return -> ResponseEntity with custom error as the response body
* If we want to handle a **generic exception** type change in the method argument to be **Exception** object -> This will be called by any exception type
##### Example Flow
![[Pasted image 20260505005318.png|556]]


> [!NOTE] 
> This Exception handler method code is only for the specific REST controller, can't be reused by other controllers

### Global Exception Handlers

Using Spring **@ControllerAdvice** annotation

##### Example
``` java
@ControllerAdvice  
public class StudentRestExceptionHandler {  
  
    @ExceptionHandler  
    public ResponseEntity<StudentErrorResponse>     
    handleException(StudentNotFoundException ex) {  
        StudentErrorResponse error = new StudentErrorResponse(  
                HttpStatus.NOT_FOUND.value(),  
                ex.getMessage(),  
                System.currentTimeMillis()  
        );  
  
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);  
    }
```

![[Pasted image 20260505011424.png]]