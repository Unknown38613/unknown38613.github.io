*   **One-to-One**
*   **One-to-Many / Many-to-One**
*   **Many-to-Many**

***

## Project Setup (quick)

**`build.gradle` or `pom.xml`** should include:

*   Spring Web
*   Spring Data JPA
*   H2 (in-memory) or MySQL

**`application.yml` (H2 for quick run):**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=MySQL;DATABASE_TO_UPPER=false
    driverClassName: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

***

# 1) One-to-One: `User` ↔ `UserProfile`

**Scenario:** One user has exactly one profile. The **owning side** holds the foreign key (`user_profile.user_id`), or alternatively you can put the FK on the `users` table. Below is the **FK on profile** (common and clear).

### Entities

```java
// User.java
package com.example.demo.one2one;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    // mappedBy -> inverse side (non-owning). Owning side is in UserProfile.user.
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY, optional = false)
    private UserProfile profile;

    public User() {}
    public User(String email) { this.email = email; }

    // getters/setters
    public Long getId() { return id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public UserProfile getProfile() { return profile; }
    public void setProfile(UserProfile profile) {
        this.profile = profile;
        if (profile != null) profile.setUser(this); // keep both sides in sync
    }
}
```

```java
// UserProfile.java
package com.example.demo.one2one;

import jakarta.persistence.*;

@Entity
@Table(name = "user_profile")
public class UserProfile {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String fullName;
    private String phone;

    // Owning side with FK user_id in user_profile
    @OneToOne
    @JoinColumn(name = "user_id", unique = true, nullable = false)
    private User user;

    public UserProfile() {}
    public UserProfile(String fullName, String phone) {
        this.fullName = fullName; this.phone = phone;
    }

    // getters/setters
    public Long getId() { return id; }
    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }

    public User getUser() { return user; }
    public void setUser(User user) { this.user = user; }
}
```

**Notes**

*   `UserProfile` is **owning side** (has `@JoinColumn`).
*   `User.profile` uses `mappedBy = "user"` and `cascade = ALL` so persisting user cascades to profile.

***

# 2) One-to-Many / Many-to-One: `Order` ↔ `OrderItem`

**Scenario:** An order has many order items. Each order item belongs to one order. The **owning side is `OrderItem`** (has the FK `order_id`).

### Entities

```java
// Order.java
package com.example.demo.one2many;

import jakarta.persistence.*;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity @Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerEmail;
    private LocalDateTime createdAt = LocalDateTime.now();

    // inverse side; items own the relationship
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    public Order() {}
    public Order(String customerEmail) { this.customerEmail = customerEmail; }

    // helpers to keep both sides in sync
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }

    // getters/setters
    public Long getId() { return id; }
    public String getCustomerEmail() { return customerEmail; }
    public void setCustomerEmail(String customerEmail) { this.customerEmail = customerEmail; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public List<OrderItem> getItems() { return items; }
}
```

```java
// OrderItem.java
package com.example.demo.one2many;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity @Table(name = "order_item")
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String sku;
    private int quantity;
    private BigDecimal unitPrice;

    // Owning side with FK order_id
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id")
    private Order order;

    public OrderItem() {}
    public OrderItem(String sku, int quantity, BigDecimal unitPrice) {
        this.sku = sku; this.quantity = quantity; this.unitPrice = unitPrice;
    }

    // getters/setters
    public Long getId() { return id; }
    public String getSku() { return sku; }
    public void setSku(String sku) { this.sku = sku; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public void setUnitPrice(BigDecimal unitPrice) { this.unitPrice = unitPrice; }
    public Order getOrder() { return order; }
    public void setOrder(Order order) { this.order = order; }
}
```

**Notes**

*   `OrderItem` is owning side with `@ManyToOne` and `@JoinColumn("order_id")`.
*   `Order` uses `mappedBy="order"`, `cascade = ALL`, and `orphanRemoval=true` to auto-delete removed items.

***

# 3) Many-to-Many: `Student` ↔ `Course`

**Scenario:** Students enroll in multiple courses; courses have many students. The **owning side** can be either; here we make `Student` the owning side and define a **join table** `student_course`.

### Entities

```java
// Student.java
package com.example.demo.many2many;

import jakarta.persistence.*;
import java.util.HashSet;
import java.util.Set;

@Entity @Table(name = "student")
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Owning side defines the join table
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    public Student() {}
    public Student(String name) { this.name = name; }

    public void enroll(Course course) {
        courses.add(course);
        course.getStudents().add(this); // sync inverse side
    }
    public void drop(Course course) {
        courses.remove(course);
        course.getStudents().remove(this);
    }

    // getters/setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Set<Course> getCourses() { return courses; }
}
```

```java
// Course.java
package com.example.demo.many2many;

import jakarta.persistence.*;
import java.util.HashSet;
import java.util.Set;

@Entity @Table(name = "course")
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    // inverse side (mappedBy)
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();

    public Course() {}
    public Course(String title) { this.title = title; }

    // getters/setters
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public Set<Student> getStudents() { return students; }
}
```

**Notes**

*   Join table `student_course(student_id, course_id)` is managed by the owning side (`Student`).
*   Many-to-many often benefits from converting to **two one-to-manys** with an explicit link entity (e.g., `Enrollment`) if you need extra fields like `enrolledOn`, `grade`.

***

## Repositories (Spring Data JPA)

```java
package com.example.demo.one2one;
import org.springframework.data.jpa.repository.JpaRepository;
public interface UserRepository extends JpaRepository<User, Long> {}

package com.example.demo.one2many;
import org.springframework.data.jpa.repository.JpaRepository;
public interface OrderRepository extends JpaRepository<Order, Long> {}

package com.example.demo.many2many;
import org.springframework.data.jpa.repository.JpaRepository;
public interface StudentRepository extends JpaRepository<Student, Long> {}
package com.example.demo.many2many;
import org.springframework.data.jpa.repository.JpaRepository;
public interface CourseRepository extends JpaRepository<Course, Long> {}
```

***

## Quick Demo Runner

```java
package com.example.demo;

import com.example.demo.one2one.*;
import com.example.demo.one2many.*;
import com.example.demo.many2many.*;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;

@Component
public class DemoData {
    @Bean
    CommandLineRunner seed(
        UserRepository userRepo,
        OrderRepository orderRepo,
        StudentRepository studentRepo,
        CourseRepository courseRepo
    ) {
        return args -> {
            // --- 1) One-to-One
            User u = new User("alice@example.com");
            u.setProfile(new UserProfile("Alice Doe", "+91-9999999999"));
            userRepo.save(u); // cascades to profile

            // --- 2) One-to-Many / Many-to-One
            Order order = new Order("bob@example.com");
            order.addItem(new OrderItem("SKU-101", 2, new BigDecimal("499.00")));
            order.addItem(new OrderItem("SKU-202", 1, new BigDecimal("1299.00")));
            orderRepo.save(order); // cascades to items

            // --- 3) Many-to-Many
            Student s1 = new Student("Rahul");
            Student s2 = new Student("Priya");
            Course c1 = new Course("Data Structures");
            Course c2 = new Course("Operating Systems");

            // enroll
            s1.enroll(c1);
            s1.enroll(c2);
            s2.enroll(c1);

            // save owning side first is fine; JPA handles join table
            courseRepo.save(c1); courseRepo.save(c2);
            studentRepo.save(s1); studentRepo.save(s2);
        };
    }
}
```

***

## Common Pitfalls & Tips

*   **Owning side matters**: Only the owning side updates the FK / join table.
*   **`mappedBy`** goes on the **inverse** side.
*   **Fetch types**: `@OneToOne` and `@ManyToOne` default to **EAGER** in JPA; many projects change to **LAZY** for performance.
*   **Cascade**: Use `CascadeType.ALL` when parent should persist/remove children automatically. Be careful with `REMOVE` in many-to-many (could delete shared entities).
*   **Orphans**: `orphanRemoval=true` only on one-to-one/one-to-many where you want deleted children when removed from the collection.
*   **Bidirectional synchronization**: Provide helper methods (`addItem`, `enroll`) to keep both sides in sync.
If you want, I can wrap this into a **minimal Spring Boot repo structure** or switch to **unidirectional** examples (simpler for beginners). Also, since you’ve been practicing with Java streams and Selenium, I can add **DTOs + controllers** to return these relationships as JSON for Postman testing. Would you like that next?
