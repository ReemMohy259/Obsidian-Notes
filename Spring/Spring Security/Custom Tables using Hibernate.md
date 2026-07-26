If you want to use Spring Data JPA / Hibernate ORM instead of plain JDBC queries, then you typically stop using:

```java id="b4q7nm"
JdbcUserDetailsManager
```

and instead create:

1. JPA entities (`User`, `Role`)
2. Repository interfaces
3. Custom `UserDetailsService`

This is the modern and most common production approach.

---
### Architecture

Instead of Spring querying tables directly via SQL:

```text id="k5v1rx"
JdbcUserDetailsManager
```

you let JPA/Hibernate load users:

```text id="v7n3sa"
DB -> JPA Repository -> UserDetailsService -> Spring Security
```

---
### 1. User Entity

Example:

```java id="w1e9ql"
@Entity
@Table(name = "members")
public class User {

    @Id
    @Column(name = "user_id")
    private String username;

    @Column(name = "pw")
    private String password;

    @Column(name = "active")
    private boolean enabled;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
            name = "roles",
            joinColumns = @JoinColumn(name = "user_id"),
            inverseJoinColumns = @JoinColumn(name = "role")
    )
    private List<Role> roles;

    // getters/setters
}
```

---
### 2. Role Entity

```java id="f9m2zp"
@Entity
@Table(name = "roles")
public class Role {

    @Id
    private String role;

    // getters/setters
}
```

In real apps you usually use numeric IDs, but this keeps example simple.

---
### 3. Repository

```java id="h7x4dc"
@Repository
public interface UserRepository extends JpaRepository<User, String> {

    Optional<User> findByUsername(String username);
}
```

---
### 4. Custom UserDetailsService

This is the key part.

Spring Security uses `UserDetailsService` to load users during login.

```java id="j3s8vy"
@Service
public class JpaUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public JpaUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        User user = userRepository.findByUsername(username)
                .orElseThrow(() ->
                        new UsernameNotFoundException(
                                "User not found"));

        List<GrantedAuthority> authorities =
                user.getRoles()
                        .stream()
                        .map(role ->
                                new SimpleGrantedAuthority(role.getRole()))
                        .toList();

        return new org.springframework.security.core.userdetails.User(
                user.getUsername(),
                user.getPassword(),
                user.isEnabled(),
                true,
                true,
                true,
                authorities
        );
    }
}
```

---
### 5. Security Config

Now you do NOT use:

```java id="k6u0re"
JdbcUserDetailsManager
```

Instead:

```java 
@Bean  
public DaoAuthenticationProvider authenticationProvider(UserService userService) {  
    DaoAuthenticationProvider auth = new DaoAuthenticationProvider();
    //set the custom user details service  
    auth.setUserDetailsService(userService); 
      
    //set the password encoder - bcrypt  
    auth.setPasswordEncoder(passwordEncoder()); 
    
    return auth;  
}
```


> [!NOTE] Note
> Modern Spring Boot often doesn't require explicitly defining this bean if:
> *  you already define `UserDetailsService`
> * you already define `PasswordEncoder`
>   
>**Spring can auto-configure it.**

---
### Important Note

When using roles:

Spring expects authorities like:

```text id="a8z5nr"
ROLE_ADMIN
ROLE_MANAGER
```

So either:

* store full names in DB
* or prepend `"ROLE_"` manually

Example:

```java id="z0x3pf"
new SimpleGrantedAuthority("ROLE_" + role.getName())
```
