### What is meant by REST
==A RESTful API is an architectural approach built on top of the HTTP protocol for data exchange. It improves interoperability between systems by enabling communication in multiple formats such as JSON, XML, and others.==

![[Pasted image 20260504215450.png|383]]

> **Note**
> Client and server can be written in any language (C#, Java, etc.) 
> In our case server is written in Java and most of the time we will use **Postman** as a client

---
### REST VS. RESTful APIs

| Aspect             | REST                                                                                                       | RESTful API                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Definition         | REST (Representational State Transfer) is an **architectural style** for designing networked applications. | A RESTful API is an **API that follows REST principles**.                              |
| Nature             | It’s a **set of constraints and guidelines**, not a protocol or standard.                                  | It’s an **implementation** of REST using HTTP.                                         |
| Usage              | Describes how systems should communicate over a network.                                                   | Used to **build web services** that follow REST rules.                                 |
| Compliance         | Can be partially followed or loosely interpreted.                                                          | Must **strictly adhere** to REST constraints (statelessness, uniform interface, etc.). |
| Focus              | Focuses on concepts like resources, representations, and statelessness.                                    | Focuses on **practical endpoints**, URLs, and HTTP methods.                            |
| Example            | Concept: Use resources like `/users` and HTTP methods like GET, POST.                                      | Actual API: `GET /users`, `POST /users`, `DELETE /users/1`.                            |
| Dependency on HTTP | Not strictly tied to HTTP (though commonly used with it).                                                  | Typically **implemented over HTTP**.                                                   |
| Goal               | Provides a theoretical foundation.                                                                         | Provides a **usable, real-world API**.                                                 |

---
### REST APIs in Spring Boot

**[[2- Spring Boot REST Controller]]**

---
### Exception Handling

**[[3- Exception Handling in Spring REST Controller]]**

---
### Known Architecture as follows
* DAO/Repository layer
* Service Layer
* Controller (REST) Layer
##### Example
```java
@Repository
public class EmployeeDAOImpl implements EmployeeDAO{

}
--------------------------------------------------------
@Service
public class EmployeeServiceImpl implements EmployeeService{
	// Spring will scans for a class that implements this interface
	private EmployeeDAO employeeDAO;
}
--------------------------------------------------------
@RestController
public class EmployeeRestController{
	// Spring will inject EmployeeServiceImpl here
	private EmployeeService employeeService; 
}
```

> [!NOTE] The Real Issue
> These layers are handling the logic for the Employee resource, what if we want to add extra resources like Customers, Orders, etc., `too many files with more boilerplate code`.
> Because the only change in this case is the **Resource Type and the ID Type**
>

* **I wish to:**
	* Create a DAO for me
	* Plug in my entity type and primary key 
	* Give me all of the basic CRUD features for free
##### The solution is:
**4- [[Spring Data JPA]]**

---
### New Architecture Becomes:
* DAO/Repository layer **(almost no code)**
* Service Layer **(delegates it's operation to repository layer)**
* Controller (REST) Layer **(exposes CRUD endpoints e.g. /api/employees)**
##### Example
```java
@Repository
public interface EmployeeReposirory extends JpaRepository<Employee,Integer>{

}
--------------------------------------------------------
@Service
public class EmployeeServiceImpl implements EmployeeService{
	// Spring will scans for a class that implements this interface
	private EmployeeReposirory employeeReposirory;
	
	@Override
	public List<Employee> findAll(){
		return employeeReposirory.findAll();
	}
}
--------------------------------------------------------
@RestController
@RequestMapping("/api") -> Base path 
public class EmployeeRestController{
	// Spring will inject EmployeeServiceImpl here
	private EmployeeService employeeService; 
	
	@GetMapping("/employees)
	public List<Employee> findAll(){
		return employeeService.findAll();
	}
}
```


> [!NOTE] Another Issue
> We still have too many **boilerplate code** in both service and controller layer

* **I wish we could tell Spring to:** 
	* Create a REST API for me 
	* Use my existing JpaRepository (entity, primary key) 
	* Give me all of the basic REST API CRUD features for free
##### The solution is:
**[[5-Spring Data REST]]**
