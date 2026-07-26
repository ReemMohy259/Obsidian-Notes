Create a DAO and just plug in your **entity type and primary key**, then spring will give you a **CRUD** implementation for **FREE** …. like MAGIC!!, which helps to minimize boiler-plate DAO code.
### JpaRepository Interface
Spring Data JPA provides the interface: JpaRepository, that exposes methods for basic CRUD.
##### Example
```java
public interface EmployeeRepository extends JpaRepository<Employee, Integer>{  
    // No extra work needed (all CRUD operations available for free)  
}
```
When spring sees this interface a lot happens internally
#### 1- Spring Scans Repository Interfaces
Spring scans packages looking for interfaces extending:
- `Repository`
- `CrudRepository`
- `JpaRepository`
#### 2- Spring Creates a Proxy Object
Since we only wrote the interface, no implementation class written by us **(don't confuse and think that spring will inject this interface, injection done only for concrete classes)**
Spring dynamically generates one at runtime, actually creates a **proxy** object.
* This proxy of type `JdkSimpleProky` as we are coding to interfaces

```java
class EmployeeRepositoryImpl implements EmployeeRepository {
    
    @Override
    public List<Employee> findAll() {
        // generated implementation
    }

    @Override
    public Employee save(Employee e) {
        // generated implementation
    }
}
```
#### 3- Proxy Delegates to SimpleJpaRepository
The generated repository implementation internally uses `SimpleJpaRepository` This is the default implementation Spring Data JPA provides. This is the class that uses the `EntityManager` 

> [!NOTE] Note
> `SimpleJpaRepository` internally handles the `@Transactional` so no need to add it on the service methods

```java
Your Code
   ↓
EmployeeRepository (interface extends JpaRepository)
   ↓
Spring Data JPA creates a Proxy implementation
   ↓
SimpleJpaRepository<Employee, Integer>  ← actual implementation
   ↓
EntityManager
   ↓
Hibernate (JPA Provider)
   ↓
Database
```

---
### Using Custom Methods Beyond CRUD

`JpaRepository` gives CRUD methods for free, but in real projects we often need custom queries like:
- Find employees by department
- Find active users
- Search by email
- Complex joins/filtering
Spring Data JPA supports this in **3 ways**.

---
#### 1- Query Method Derivation (Method Name Parsing)

Spring can generate queries just from method names.
##### Example

```java
public interface EmployeeRepository extends JpaRepository<Employee, Integer> {

    Employee findByEmail(String email);

    List<Employee> findByDepartment(String department);

    List<Employee> findBySalaryGreaterThan(double salary);
}
```

Spring reads the method names and internally generates SQL/JPQL.
##### Internally

```text
findByEmail()
      ↓
Spring parses method name
      ↓
Creates JPQL query dynamically
      ↓
EntityManager executes query
      ↓
Hibernate generates SQL
      ↓
Database
```

Example generated query:

```sql
SELECT e FROM Employee e WHERE e.email = :email
```

So we still write **no implementation code**.
##### Nested Property Traversal

* **Direct Property Example**
```java
List<Person> findByAddressZipCode(ZipCode zipCode);
```
- **We actually want to traverses this** `person.address.zipCode`

==Property Resolution Algorithm==
Spring Data tries to resolve **property names** by:
1. Checking **full property name** first.
2. Splitting **camel-case parts** from **right to left**.

**Example:**
```text
AddressZipCode
→ AddressZip + Code 
→ Address + ZipCode
```

**Ambiguity Problem**
If `Person` has `addressZip`. Spring may incorrectly resolve `AddressZip + Code` and fail.

> [!NOTE] Using `_` to Remove Ambiguity
> Use `_` to explicitly define traversal points. 
> **Example**:
> `findByAddress_ZipCode(ZipCode zipCode)` This clearly means `address.zipCode`

**Important Note:**
- `_` is treated as a reserved character.
- Prefer **camelCase** naming in Java fields.
- Avoid underscores in property names.
---
#### 2- Using `@Query`
For more complex queries, we can write JPQL or native SQL manually.
##### JPQL Example
```java
public interface EmployeeRepository extends JpaRepository<Employee, Integer> {

    @Query("SELECT e FROM Employee e WHERE e.salary > :salary")
    List<Employee> getHighSalaryEmployees(
            @Param("salary") double salary);
}
```
##### Native SQL Example
```java
@Query(
    value = "SELECT * FROM employee WHERE department = :dept",
    nativeQuery = true
)
List<Employee> findByDept(@Param("dept") String dept);
```

##### Internally:
```text
Repository Proxy
      ↓
Reads @Query annotation
      ↓
Creates query using EntityManager
      ↓
Hibernate executes SQL
      ↓
Database
```

> `@Query` expects a `SELECT` query but if we want to use other updating queries we have to annotate the method with **`@Modifying`** and **`@Transactional`**
---
#### 3- Custom Repository Implementation (Manual Logic)
Sometimes query methods are not enough.
Example:
- Dynamic queries
- Criteria API   
- Complex joins
- Stored procedures
- Multiple entity operations
Then we create a custom repository implementation.
##### Step 1 — Create Custom Repository Interface

```java
public interface EmployeeCustomRepository {
    List<Employee> findEmployeesWithCustomLogic();
}
```
##### Step 2 — Implement It

> **Naming is IMPORTANT**  
> Spring expects `RepositoryName + Impl`

```java
@Repository
public class EmployeeCustomRepositoryImpl implements EmployeeCustomRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public List<Employee> findEmployeesWithCustomLogic() {

        String jpql =
            "SELECT e FROM Employee e WHERE e.salary > 10000";

        return entityManager
                .createQuery(jpql, Employee.class)
                .getResultList();
    }
}
```
##### Step 3 — Extend Both Interfaces

```java
public interface EmployeeRepository
        extends JpaRepository<Employee, Integer>,
                EmployeeCustomRepository {
}
```

Now `EmployeeRepository` contains:
- Built-in CRUD methods
- Query-derived methods
- Custom implemented methods
##### Internal Flow with Custom Repository

```text
employeeRepository.findEmployeesWithCustomLogic()
        ↓
Repository Proxy
        ↓
EmployeeCustomRepositoryImpl
        ↓
EntityManager
        ↓
Hibernate
        ↓
Database
```

`EmployeeCustomRepositoryImpl` is itself a normal Spring bean. BUT it is usually **not injected directly**. Instead Spring creates `Proxy(EmployeeRepository)` that internally contains references to:
- `SimpleJpaRepository`
- `EmployeeCustomRepositoryImpl`

**Mental Model:**
```java
class EmployeeRepositoryProxy implements EmployeeRepository {
    private SimpleJpaRepository target;
    private EmployeeCustomRepositoryImpl customTarget;
}
```
---
### Different Spring Data Interfaces

We mainly have **2 main packages** related to spring data `org.springframework.data.jpa.repository` and `org.springframework.data.repository`

> As we have 2 different types of DB (relational DB and non-relational DB). 
> So, all interfaces inside `data.jpa` package are related to **SQL DB** (e.g. MySQL, PostgreSQL).
> But, for the other package are more general can be used for relational and non-relational DB like **MongoDB** 

For interfaces inside the general package spring will generate the implementation based on:
* Known from the **entity type**, what annotations used on the entity class `@Entity` or `@Document`
* OR, **explicitly** defined in a configuration class
```Java
@EnableJpaRepositories(basePackages = "com.acme.repositories.jpa")   
@EnableMongoRepositories(basePackages = "com.acme.repositories.mongo") 
class Configuration {
}
```

##### Different Interfaces:
* `Repository<T, ID>` -> "**Markable interface**" Basic repository interface any method needed must be defined inside it
* `CrudRepository<T, ID>` -> Extends `Repository` but it exposes automatically all **CRUD** operations
* `ListCrudRepository<T, ID>` -> Extends `CrudRepository` but its return type is **List** instead of **Iterable**
* `PagingAndSortingRepository<T, ID>` -> Extends `Repository` but it exposes methods for pagination and sorting the result set returned by `findAll` method
* `ListPagingAndSortingRepository<T, ID>` -> Extends `PagingAndSortingRepository` but its return type is **List** instead of **Iterable**
* `JpaRepository<T, ID>` -> Extends `ListCrudRepository` and `ListPagingAndSortingRepository`

> [!NOTE] Note
>Spring Data scans repository interfaces and automatically creates implementation classes and beans for them at runtime.  
>
>Interfaces such as `CrudRepository`, and `JpaRepository` are annotated with `@NoRepositoryBean` to tell Spring **not** to create beans directly for these base interfaces.  
>
>Instead, Spring should only create beans for concrete repository interfaces defined by the developer.
