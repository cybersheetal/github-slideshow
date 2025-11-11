# Many-to-Many Relationship Setup in Spring Boot - Quick Reference Guide

This guide provides a comprehensive overview of implementing many-to-many relationships in Spring Boot applications using JPA and Hibernate.

## Table of Contents
1. [Introduction](#introduction)
2. [Database Design](#database-design)
3. [Entity Setup](#entity-setup)
4. [Repository Layer](#repository-layer)
5. [Service Layer](#service-layer)
6. [Controller Layer](#controller-layer)
7. [Configuration](#configuration)
8. [Best Practices](#best-practices)
9. [Common Pitfalls](#common-pitfalls)
10. [Testing](#testing)

## Introduction

A **many-to-many relationship** occurs when multiple records in one table are associated with multiple records in another table. Common examples include:

- **Students ↔ Courses**: A student can enroll in many courses, and a course can have many students
- **Authors ↔ Books**: An author can write many books, and a book can have multiple authors
- **Users ↔ Roles**: A user can have many roles, and a role can be assigned to many users

## Database Design

In a relational database, many-to-many relationships require a **join table** (also called junction or bridge table):

```
┌──────────┐        ┌──────────────────┐        ┌──────────┐
│ Student  │        │ student_course   │        │ Course   │
├──────────┤        ├──────────────────┤        ├──────────┤
│ id (PK)  │───────<│ student_id (FK)  │>───────│ id (PK)  │
│ name     │        │ course_id (FK)   │        │ name     │
│ email    │        └──────────────────┘        │ credits  │
└──────────┘                                    └──────────┘
```

## Entity Setup

### Student Entity (Owner Side)

```java
package com.example.demo.entity;

import javax.persistence.*;
import java.util.HashSet;
import java.util.Set;

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
}
```

### Course Entity (Inverse Side)

```java
package com.example.demo.entity;

import javax.persistence.*;
import java.util.HashSet;
import java.util.Set;

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
    @JsonIgnore  // Prevents circular reference during JSON serialization
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
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Course)) return false;
        Course course = (Course) o;
        return Objects.equals(getId(), course.getId());
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(getId());
    }
}
```

### Key Annotations Explained

| Annotation | Description |
|------------|-------------|
| `@ManyToMany` | Defines a many-to-many relationship between two entities |
| `@JoinTable` | Specifies the join table details (only on owner side) |
| `mappedBy` | Indicates the inverse side of the relationship |
| `@JsonIgnore` | Prevents circular reference during JSON serialization |

## Repository Layer

### StudentRepository.java

```java
package com.example.demo.repository;

import com.example.demo.entity.Student;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
    
    Optional<Student> findByEmail(String email);
    
    @Query("SELECT s FROM Student s JOIN FETCH s.courses WHERE s.id = :id")
    Optional<Student> findByIdWithCourses(@Param("id") Long id);
    
    @Query("SELECT DISTINCT s FROM Student s JOIN FETCH s.courses")
    List<Student> findAllWithCourses();
}
```

### CourseRepository.java

```java
package com.example.demo.repository;

import com.example.demo.entity.Course;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface CourseRepository extends JpaRepository<Course, Long> {
    
    Optional<Course> findByName(String name);
    
    @Query("SELECT c FROM Course c JOIN FETCH c.students WHERE c.id = :id")
    Optional<Course> findByIdWithStudents(@Param("id") Long id);
    
    @Query("SELECT DISTINCT c FROM Course c JOIN FETCH c.students")
    List<Course> findAllWithStudents();
}
```

## Service Layer

### StudentService.java

```java
package com.example.demo.service;

import com.example.demo.entity.Course;
import com.example.demo.entity.Student;
import com.example.demo.repository.CourseRepository;
import com.example.demo.repository.StudentRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

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
            .orElseThrow(() -> new ResourceNotFoundException("Student not found with id: " + studentId));
        
        Course course = courseRepository.findById(courseId)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found with id: " + courseId));
        
        student.addCourse(course);
        return studentRepository.save(student);
    }
    
    public Student unenrollStudentFromCourse(Long studentId, Long courseId) {
        Student student = studentRepository.findById(studentId)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found with id: " + studentId));
        
        Course course = courseRepository.findById(courseId)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found with id: " + courseId));
        
        student.removeCourse(course);
        return studentRepository.save(student);
    }
    
    @Transactional(readOnly = true)
    public Student getStudentWithCourses(Long id) {
        return studentRepository.findByIdWithCourses(id)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found with id: " + id));
    }
    
    @Transactional(readOnly = true)
    public List<Student> getAllStudents() {
        return studentRepository.findAll();
    }
    
    public void deleteStudent(Long id) {
        Student student = studentRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Student not found with id: " + id));
        studentRepository.delete(student);
    }
}
```

### CourseService.java

```java
package com.example.demo.service;

import com.example.demo.entity.Course;
import com.example.demo.repository.CourseRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Transactional
public class CourseService {
    
    @Autowired
    private CourseRepository courseRepository;
    
    public Course createCourse(Course course) {
        return courseRepository.save(course);
    }
    
    @Transactional(readOnly = true)
    public Course getCourseWithStudents(Long id) {
        return courseRepository.findByIdWithStudents(id)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found with id: " + id));
    }
    
    @Transactional(readOnly = true)
    public List<Course> getAllCourses() {
        return courseRepository.findAll();
    }
    
    public void deleteCourse(Long id) {
        Course course = courseRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Course not found with id: " + id));
        courseRepository.delete(course);
    }
}
```

## Controller Layer

### StudentController.java

```java
package com.example.demo.controller;

import com.example.demo.entity.Student;
import com.example.demo.service.StudentService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

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
    
    @GetMapping
    public ResponseEntity<List<Student>> getAllStudents() {
        List<Student> students = studentService.getAllStudents();
        return ResponseEntity.ok(students);
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
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteStudent(@PathVariable Long id) {
        studentService.deleteStudent(id);
        return ResponseEntity.noContent().build();
    }
}
```

### CourseController.java

```java
package com.example.demo.controller;

import com.example.demo.entity.Course;
import com.example.demo.service.CourseService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/courses")
public class CourseController {
    
    @Autowired
    private CourseService courseService;
    
    @PostMapping
    public ResponseEntity<Course> createCourse(@RequestBody Course course) {
        Course created = courseService.createCourse(course);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Course> getCourse(@PathVariable Long id) {
        Course course = courseService.getCourseWithStudents(id);
        return ResponseEntity.ok(course);
    }
    
    @GetMapping
    public ResponseEntity<List<Course>> getAllCourses() {
        List<Course> courses = courseService.getAllCourses();
        return ResponseEntity.ok(courses);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteCourse(@PathVariable Long id) {
        courseService.deleteCourse(id);
        return ResponseEntity.noContent().build();
    }
}
```

## Configuration

### application.properties (Development with H2)

```properties
# Application Name
spring.application.name=many-to-many-demo

# Database Configuration (H2)
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# JPA/Hibernate Configuration
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# H2 Console
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

# Server Configuration
server.port=8080
```

### application-prod.properties (Production with PostgreSQL)

```properties
# Database Configuration (PostgreSQL)
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=postgres
spring.datasource.password=yourpassword

# JPA/Hibernate Configuration
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false

# Connection Pool
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
```

### Maven Dependencies (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.14</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>many-to-many-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>many-to-many-demo</name>
    <description>Demo project for Many-to-Many Relationship</description>
    
    <properties>
        <java.version>11</java.version>
    </properties>
    
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
        
        <!-- Spring Boot Starter Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <!-- Lombok (optional) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## Best Practices

### 1. Use Set instead of List

```java
// ✅ Good - Better performance, prevents duplicates
private Set<Course> courses = new HashSet<>();

// ❌ Avoid - Can have duplicates, slower operations
private List<Course> courses = new ArrayList<>();
```

### 2. Implement equals() and hashCode()

Always implement these methods based on the primary key:

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

### 3. Use Helper Methods for Bidirectional Consistency

```java
// Always maintain both sides of the relationship
public void addCourse(Course course) {
    this.courses.add(course);
    course.getStudents().add(this);
}

public void removeCourse(Course course) {
    this.courses.remove(course);
    course.getStudents().remove(this);
}
```

### 4. Choose Cascade Types Carefully

```java
// ❌ Dangerous - Deleting a student will delete all courses!
@ManyToMany(cascade = CascadeType.ALL)

// ✅ Safe - Only cascades persist and merge operations
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
```

### 5. Use Fetch Type Wisely

```java
// Default is LAZY - usually the best choice
@ManyToMany(fetch = FetchType.LAZY)

// Use EAGER only when always needed (can cause performance issues)
@ManyToMany(fetch = FetchType.EAGER)
```

### 6. Use JOIN FETCH to Avoid N+1 Queries

```java
@Query("SELECT s FROM Student s JOIN FETCH s.courses WHERE s.id = :id")
Optional<Student> findByIdWithCourses(@Param("id") Long id);
```

## Common Pitfalls

### 1. LazyInitializationException

**Problem:** Accessing lazy-loaded collection outside of a transaction.

```java
// ❌ This will fail if courses are not loaded
student.getCourses().size();
```

**Solution:** Use JOIN FETCH or ensure you're within a transaction:

```java
// ✅ Load eagerly when needed
@Query("SELECT s FROM Student s JOIN FETCH s.courses")
List<Student> findAllWithCourses();
```

### 2. N+1 Query Problem

**Problem:** Hibernate executes one query to load entities and N queries to load associations.

**Solution:** Use JOIN FETCH or Entity Graphs:

```java
@Query("SELECT DISTINCT s FROM Student s LEFT JOIN FETCH s.courses")
List<Student> findAllWithCourses();
```

### 3. Circular JSON Reference

**Problem:** Bidirectional relationships cause infinite recursion during JSON serialization.

**Solutions:**

a) Use @JsonIgnore on one side:
```java
@ManyToMany(mappedBy = "courses")
@JsonIgnore
private Set<Student> students = new HashSet<>();
```

b) Use DTOs (recommended):
```java
public class StudentDTO {
    private Long id;
    private String name;
    private Set<CourseSimpleDTO> courses;
}
```

### 4. Not Maintaining Bidirectional Consistency

**Problem:** Only setting one side of the relationship.

```java
// ❌ Wrong - only one side updated
student.getCourses().add(course);
```

**Solution:** Use helper methods:

```java
// ✅ Correct - both sides updated
student.addCourse(course);
```

## Testing

### Unit Test Example

```java
package com.example.demo.service;

import com.example.demo.entity.Course;
import com.example.demo.entity.Student;
import com.example.demo.repository.CourseRepository;
import com.example.demo.repository.StudentRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Transactional
class StudentServiceTest {
    
    @Autowired
    private StudentService studentService;
    
    @Autowired
    private CourseRepository courseRepository;
    
    @Autowired
    private StudentRepository studentRepository;
    
    @Test
    void testEnrollStudentInCourse() {
        // Given
        Student student = new Student("John Doe", "john@example.com");
        student = studentRepository.save(student);
        
        Course course = new Course("Spring Boot Fundamentals", 3);
        course = courseRepository.save(course);
        
        // When
        Student enrolled = studentService.enrollStudentInCourse(
            student.getId(), 
            course.getId()
        );
        
        // Then
        assertThat(enrolled.getCourses()).hasSize(1);
        assertThat(enrolled.getCourses()).contains(course);
        
        // Verify bidirectional relationship
        Course savedCourse = courseRepository.findById(course.getId()).get();
        assertThat(savedCourse.getStudents()).contains(student);
    }
    
    @Test
    void testUnenrollStudentFromCourse() {
        // Given
        Student student = new Student("Jane Smith", "jane@example.com");
        Course course = new Course("Advanced Java", 4);
        
        student = studentRepository.save(student);
        course = courseRepository.save(course);
        
        student = studentService.enrollStudentInCourse(student.getId(), course.getId());
        assertThat(student.getCourses()).hasSize(1);
        
        // When
        Student unenrolled = studentService.unenrollStudentFromCourse(
            student.getId(), 
            course.getId()
        );
        
        // Then
        assertThat(unenrolled.getCourses()).isEmpty();
        
        // Verify bidirectional relationship
        Course savedCourse = courseRepository.findById(course.getId()).get();
        assertThat(savedCourse.getStudents()).doesNotContain(student);
    }
}
```

## API Usage Examples

### Create a Student

```bash
POST http://localhost:8080/api/students
Content-Type: application/json

{
  "name": "Alice Johnson",
  "email": "alice@example.com"
}

# Response (201 Created)
{
  "id": 1,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "courses": []
}
```

### Create a Course

```bash
POST http://localhost:8080/api/courses
Content-Type: application/json

{
  "name": "Spring Boot Fundamentals",
  "credits": 3
}

# Response (201 Created)
{
  "id": 1,
  "name": "Spring Boot Fundamentals",
  "credits": 3
}
```

### Enroll Student in Course

```bash
POST http://localhost:8080/api/students/1/courses/1

# Response (200 OK)
{
  "id": 1,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "courses": [
    {
      "id": 1,
      "name": "Spring Boot Fundamentals",
      "credits": 3
    }
  ]
}
```

### Get Student with Courses

```bash
GET http://localhost:8080/api/students/1

# Response (200 OK)
{
  "id": 1,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "courses": [
    {
      "id": 1,
      "name": "Spring Boot Fundamentals",
      "credits": 3
    },
    {
      "id": 2,
      "name": "Microservices Architecture",
      "credits": 4
    }
  ]
}
```

### Unenroll Student from Course

```bash
DELETE http://localhost:8080/api/students/1/courses/1

# Response (200 OK)
{
  "id": 1,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "courses": [
    {
      "id": 2,
      "name": "Microservices Architecture",
      "credits": 4
    }
  ]
}
```

## Advanced: Many-to-Many with Extra Columns

When you need additional data in the join table (e.g., enrollment date, grade), use an intermediate entity:

### EnrollmentId (Composite Key)

```java
@Embeddable
public class EnrollmentId implements Serializable {
    
    @Column(name = "student_id")
    private Long studentId;
    
    @Column(name = "course_id")
    private Long courseId;
    
    // Constructors, getters, setters, equals, hashCode
}
```

### Enrollment Entity

```java
@Entity
@Table(name = "enrollments")
public class Enrollment {
    
    @EmbeddedId
    private EnrollmentId id = new EnrollmentId();
    
    @ManyToOne
    @MapsId("studentId")
    @JoinColumn(name = "student_id")
    private Student student;
    
    @ManyToOne
    @MapsId("courseId")
    @JoinColumn(name = "course_id")
    private Course course;
    
    @Column(nullable = false)
    private LocalDate enrollmentDate;
    
    @Column(length = 2)
    private String grade;
    
    // Constructors, getters, setters
}
```

### Updated Student Entity

```java
@Entity
public class Student {
    // ... existing fields
    
    @OneToMany(mappedBy = "student", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<Enrollment> enrollments = new HashSet<>();
    
    // Helper method
    public void enrollInCourse(Course course, LocalDate date) {
        Enrollment enrollment = new Enrollment();
        enrollment.setStudent(this);
        enrollment.setCourse(course);
        enrollment.setEnrollmentDate(date);
        enrollments.add(enrollment);
        course.getEnrollments().add(enrollment);
    }
}
```

## Conclusion

Implementing many-to-many relationships in Spring Boot requires careful consideration of:

1. **Entity design** with proper annotations
2. **Bidirectional consistency** using helper methods
3. **Cascade types** to avoid unintended deletions
4. **Fetch strategies** to optimize performance
5. **JSON serialization** to prevent circular references
6. **Repository queries** with JOIN FETCH to avoid N+1 problems

By following the patterns and best practices outlined in this guide, you can build robust and efficient many-to-many relationships in your Spring Boot applications.

## Resources

- [Spring Data JPA Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Hibernate Documentation](https://hibernate.org/orm/documentation/)
- [Baeldung JPA Tutorials](https://www.baeldung.com/jpa-hibernate-guide)
- [Spring Boot Reference Guide](https://docs.spring.io/spring-boot/docs/current/reference/html/)

---

**Happy Coding!** 🚀
