---
layout: slide
title: "Many-to-Many Relationship in Spring Boot"
---

## Many-to-Many Relationship Setup in Spring Boot

A comprehensive guide to implementing many-to-many relationships using JPA and Hibernate.

---

## What is a Many-to-Many Relationship?

A **many-to-many relationship** occurs when multiple records in one table are associated with multiple records in another table.

### Real-World Examples:
- **Students ↔ Courses**: A student can enroll in many courses, and a course can have many students
- **Authors ↔ Books**: An author can write many books, and a book can have multiple authors
- **Users ↔ Roles**: A user can have many roles, and a role can be assigned to many users

---

## Database Design

In a relational database, many-to-many relationships require a **join table** (also called junction or bridge table).

```
┌──────────┐        ┌──────────────────┐        ┌──────────┐
│ Student  │        │ student_course   │        │ Course   │
├──────────┤        ├──────────────────┤        ├──────────┤
│ id (PK)  │───────<│ student_id (FK)  │>───────│ id (PK)  │
│ name     │        │ course_id (FK)   │        │ name     │
│ email    │        └──────────────────┘        │ credits  │
└──────────┘                                    └──────────┘
```

---

## Spring Boot Entity Setup

### Step 1: Create the Student Entity

```java
@Entity
@Table(name = "students")
public class Student {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
    
    // Constructors
    public Student() {}
    
    public Student(String name, String email) {
        this.name = name;
        this.email = email;
    }
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    
    public Set<Course> getCourses() { return courses; }
    public void setCourses(Set<Course> courses) { this.courses = courses; }
    
    // Helper methods for bidirectional relationship
    public void addCourse(Course course) {
        this.courses.add(course);
        course.getStudents().add(this);
    }
    
    public void removeCourse(Course course) {
        this.courses.remove(course);
        course.getStudents().remove(this);
    }
}
```

---

## Step 2: Create the Course Entity

```java
@Entity
@Table(name = "courses")
public class Course {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private Integer credits;
    
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
    
    // Constructors
    public Course() {}
    
    public Course(String name, Integer credits) {
        this.name = name;
        this.credits = credits;
    }
    
    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public Integer getCredits() { return credits; }
    public void setCredits(Integer credits) { this.credits = credits; }
    
    public Set<Student> getStudents() { return students; }
    public void setStudents(Set<Student> students) { this.students = students; }
}
```

---

## Understanding Key Annotations

### @ManyToMany
Defines a many-to-many relationship between two entities.

### @JoinTable (Owner Side)
Specifies the join table details:
- **name**: Name of the join table
- **joinColumns**: Foreign key column referencing the owning entity
- **inverseJoinColumns**: Foreign key column referencing the non-owning entity

### mappedBy (Non-Owner Side)
Indicates this is the inverse side of the relationship. Points to the field name in the owning entity.

### CascadeType
- **PERSIST**: Save related entities when saving parent
- **MERGE**: Update related entities when updating parent
- **REMOVE**: Delete related entities when deleting parent (⚠️ Use carefully!)
- **ALL**: All cascade operations

---

## Repository Layer

### Student Repository

```java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
    
    Optional<Student> findByEmail(String email);
    
    @Query("SELECT s FROM Student s JOIN FETCH s.courses WHERE s.id = :id")
    Optional<Student> findByIdWithCourses(@Param("id") Long id);
    
    @Query("SELECT DISTINCT s FROM Student s JOIN FETCH s.courses")
    List<Student> findAllWithCourses();
}
```

### Course Repository

```java
@Repository
public interface CourseRepository extends JpaRepository<Course, Long> {
    
    Optional<Course> findByName(String name);
    
    @Query("SELECT c FROM Course c JOIN FETCH c.students WHERE c.id = :id")
    Optional<Course> findByIdWithStudents(@Param("id") Long id);
    
    @Query("SELECT DISTINCT c FROM Course c JOIN FETCH c.students")
    List<Course> findAllWithStudents();
}
```

---

## Service Layer

### Student Service

```java
@Service
@Transactional
public class StudentService {
    
    @Autowired
    private StudentRepository studentRepository;
    
    @Autowired
    private CourseRepository courseRepository;
    
    public Student createStudent(Student student) {
        return studentRepository.save(student);
    }
    
    public Student enrollStudentInCourse(Long studentId, Long courseId) {
        Student student = studentRepository.findById(studentId)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found"));
        
        Course course = courseRepository.findById(courseId)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found"));
        
        student.addCourse(course);
        return studentRepository.save(student);
    }
    
    public Student unenrollStudentFromCourse(Long studentId, Long courseId) {
        Student student = studentRepository.findById(studentId)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found"));
        
        Course course = courseRepository.findById(courseId)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found"));
        
        student.removeCourse(course);
        return studentRepository.save(student);
    }
    
    @Transactional(readOnly = true)
    public Student getStudentWithCourses(Long id) {
        return studentRepository.findByIdWithCourses(id)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found"));
    }
    
    @Transactional(readOnly = true)
    public List<Student> getAllStudents() {
        return studentRepository.findAll();
    }
}
```

---

## REST Controller

```java
@RestController
@RequestMapping("/api/students")
public class StudentController {
    
    @Autowired
    private StudentService studentService;
    
    @PostMapping
    public ResponseEntity<Student> createStudent(@RequestBody Student student) {
        Student created = studentService.createStudent(student);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Student> getStudent(@PathVariable Long id) {
        Student student = studentService.getStudentWithCourses(id);
        return ResponseEntity.ok(student);
    }
    
    @PostMapping("/{studentId}/courses/{courseId}")
    public ResponseEntity<Student> enrollInCourse(
            @PathVariable Long studentId,
            @PathVariable Long courseId) {
        Student updated = studentService.enrollStudentInCourse(studentId, courseId);
        return ResponseEntity.ok(updated);
    }
    
    @DeleteMapping("/{studentId}/courses/{courseId}")
    public ResponseEntity<Student> unenrollFromCourse(
            @PathVariable Long studentId,
            @PathVariable Long courseId) {
        Student updated = studentService.unenrollStudentFromCourse(studentId, courseId);
        return ResponseEntity.ok(updated);
    }
    
    @GetMapping
    public ResponseEntity<List<Student>> getAllStudents() {
        List<Student> students = studentService.getAllStudents();
        return ResponseEntity.ok(students);
    }
}
```

---

## Handling JSON Serialization

### Problem: Infinite Recursion
Many-to-many bidirectional relationships can cause infinite loops during JSON serialization.

### Solution 1: Use @JsonIgnore

```java
@Entity
public class Course {
    // ... other fields
    
    @ManyToMany(mappedBy = "courses")
    @JsonIgnore  // Prevents serialization of students in Course
    private Set<Student> students = new HashSet<>();
}
```

### Solution 2: Use @JsonManagedReference and @JsonBackReference

```java
// In Student entity (owner/managed side)
@ManyToMany
@JoinTable(...)
@JsonManagedReference
private Set<Course> courses = new HashSet<>();

// In Course entity (inverse/back side)
@ManyToMany(mappedBy = "courses")
@JsonBackReference
private Set<Student> students = new HashSet<>();
```

---

## DTOs (Data Transfer Objects)

### Best Practice: Use DTOs to control JSON output

```java
public class StudentDTO {
    private Long id;
    private String name;
    private String email;
    private Set<CourseDTO> courses;
    
    // Constructors, getters, setters
}

public class CourseDTO {
    private Long id;
    private String name;
    private Integer credits;
    // No students field to avoid circular reference
    
    // Constructors, getters, setters
}
```

### Mapper Class

```java
@Component
public class StudentMapper {
    
    public StudentDTO toDTO(Student student) {
        StudentDTO dto = new StudentDTO();
        dto.setId(student.getId());
        dto.setName(student.getName());
        dto.setEmail(student.getEmail());
        
        Set<CourseDTO> courseDTOs = student.getCourses().stream()
            .map(this::toSimpleCourseDTO)
            .collect(Collectors.toSet());
        dto.setCourses(courseDTOs);
        
        return dto;
    }
    
    private CourseDTO toSimpleCourseDTO(Course course) {
        CourseDTO dto = new CourseDTO();
        dto.setId(course.getId());
        dto.setName(course.getName());
        dto.setCredits(course.getCredits());
        return dto;
    }
}
```

---

## Application Properties Configuration

```properties
# Database Configuration (H2 for development)
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# JPA/Hibernate Configuration
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# H2 Console (for development)
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

### For Production (PostgreSQL):

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=postgres
spring.datasource.password=yourpassword
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=validate
```

---

## Maven Dependencies (pom.xml)

```xml
<dependencies>
    <!-- Spring Boot Starter Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- H2 Database (for development) -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- PostgreSQL Driver (for production) -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Lombok (optional, for reducing boilerplate) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## Best Practices

### 1. Use Set instead of List
```java
// ✅ Good - Better performance, no duplicates
private Set<Course> courses = new HashSet<>();

// ❌ Avoid - Can have duplicates, slower
private List<Course> courses = new ArrayList<>();
```

### 2. Implement equals() and hashCode()
```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Student)) return false;
    Student student = (Student) o;
    return Objects.equals(getId(), student.getId());
}

@Override
public int hashCode() {
    return Objects.hash(getId());
}
```

### 3. Use Helper Methods
```java
// Always maintain bidirectional consistency
public void addCourse(Course course) {
    this.courses.add(course);
    course.getStudents().add(this);
}
```

---

## Best Practices (Continued)

### 4. Avoid CascadeType.REMOVE
```java
// ❌ Dangerous - Deleting a student will delete all courses!
@ManyToMany(cascade = CascadeType.ALL)

// ✅ Safe - Only persist and merge cascade
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
```

### 5. Use Fetch Type Wisely
```java
// Default is LAZY for @ManyToMany - usually best
@ManyToMany(fetch = FetchType.LAZY)

// Use EAGER only when always needed
@ManyToMany(fetch = FetchType.EAGER)  // Use sparingly!
```

### 6. Handle Lazy Initialization
```java
// Use JOIN FETCH in queries to avoid LazyInitializationException
@Query("SELECT s FROM Student s JOIN FETCH s.courses WHERE s.id = :id")
Optional<Student> findByIdWithCourses(@Param("id") Long id);
```

---

## Common Pitfalls and Solutions

### Pitfall 1: LazyInitializationException
**Problem**: Accessing lazy-loaded collection outside transaction
```java
// ❌ Fails if courses not loaded
student.getCourses().size();
```

**Solution**: Use JOIN FETCH or @Transactional
```java
// ✅ Load eagerly when needed
@Query("SELECT s FROM Student s JOIN FETCH s.courses")
List<Student> findAllWithCourses();
```

### Pitfall 2: N+1 Query Problem
**Problem**: Multiple queries for collections
**Solution**: Use JOIN FETCH or Entity Graphs

### Pitfall 3: Not Maintaining Bidirectional Consistency
**Problem**: Only setting one side of relationship
**Solution**: Use helper methods like addCourse()

---

## Testing the Many-to-Many Relationship

### Unit Test Example

```java
@SpringBootTest
@Transactional
class StudentServiceTest {
    
    @Autowired
    private StudentService studentService;
    
    @Autowired
    private CourseRepository courseRepository;
    
    @Test
    void testEnrollStudentInCourse() {
        // Given
        Student student = new Student("John Doe", "john@example.com");
        student = studentService.createStudent(student);
        
        Course course = new Course("Spring Boot", 3);
        course = courseRepository.save(course);
        
        // When
        Student enrolled = studentService.enrollStudentInCourse(
            student.getId(), 
            course.getId()
        );
        
        // Then
        assertThat(enrolled.getCourses()).hasSize(1);
        assertThat(enrolled.getCourses()).contains(course);
    }
}
```

---

## Advanced: Many-to-Many with Extra Columns

Sometimes you need additional data in the join table (e.g., enrollment date, grade).

### Solution: Use an Intermediate Entity

```java
@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;
    
    @ManyToOne
    @MapsId("studentId")
    private Student student;
    
    @ManyToOne
    @MapsId("courseId")
    private Course course;
    
    @Column(nullable = false)
    private LocalDate enrollmentDate;
    
    private String grade;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    
    // equals() and hashCode()
}
```

---

## Example API Requests

### Create a Student
```bash
POST /api/students
Content-Type: application/json

{
  "name": "Alice Johnson",
  "email": "alice@example.com"
}
```

### Enroll Student in Course
```bash
POST /api/students/1/courses/2
```

### Get Student with Courses
```bash
GET /api/students/1

Response:
{
  "id": 1,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "courses": [
    {
      "id": 2,
      "name": "Spring Boot Fundamentals",
      "credits": 3
    }
  ]
}
```

---

## Summary

### Key Takeaways:
1. **Many-to-many** requires a join table in the database
2. Use **@ManyToMany** and **@JoinTable** on the owning side
3. Use **mappedBy** on the inverse side
4. Always maintain **bidirectional consistency** with helper methods
5. Use **Set** instead of List for better performance
6. Implement **equals()** and **hashCode()** properly
7. Handle **JSON serialization** carefully (use DTOs or @JsonIgnore)
8. Use **JOIN FETCH** to avoid N+1 queries
9. Be cautious with **CascadeType.REMOVE**
10. Test thoroughly to ensure data integrity

---

## Resources and Further Reading

### Official Documentation:
- [Spring Data JPA Documentation](https://spring.io/projects/spring-data-jpa)
- [Hibernate Documentation](https://hibernate.org/orm/documentation/)

### Recommended Books:
- "Pro Spring Boot 2" by Felipe Gutierrez
- "Spring in Action" by Craig Walls

### Online Resources:
- Baeldung: Spring Data JPA Tutorials
- Vlad Mihalcea's Blog on Hibernate
- Spring Boot official guides

---

## Thank You!

Questions? Feel free to reach out!

Happy coding with Spring Boot! 🚀
