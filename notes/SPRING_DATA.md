## Spring Data JPA auditing

> **In short**: `@EnableJpaAuditing` and an entity listener fill `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy` and `@LastModifiedBy` automatically, the user coming from an `AuditorAware` bean (e.g. from Spring Security). Unlike Hibernate's `@CreationTimestamp`, it also tracks who made the change.

Spring Data fills audit fields automatically: when (`@CreatedDate`, `@LastModifiedDate`) and **who** (`@CreatedBy`, `@LastModifiedBy`).

```java
@Configuration
@EnableJpaAuditing
class AuditConfig {

  @Bean
  AuditorAware<String> auditorAware() { // who is the current user
    return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
        .map(Authentication::getName);
  }
}

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {
  @CreatedDate @Column(updatable = false) private Instant createdAt;
  @LastModifiedDate private Instant updatedAt;
  @CreatedBy @Column(updatable = false) private String createdBy;
  @LastModifiedBy private String updatedBy;
}

@Entity
public class Order extends AuditableEntity { ... }
```

How it works: `AuditingEntityListener` is a JPA entity listener, called on `@PrePersist` and `@PreUpdate`; it sets the dates and asks `AuditorAware` for the user. By default, `@LastModifiedDate` / `@LastModifiedBy` are also set on creation.

To control the time in tests, declare a `DateTimeProvider` (e.g. based on a fixed `Clock`) and reference it: `@EnableJpaAuditing(dateTimeProviderRef = "auditingDateTimeProvider")`.

Compared to Hibernate's `@CreationTimestamp` / `@UpdateTimestamp` (see [Hibernate](HIBERNATE.md)):

|                       | `@CreationTimestamp` (Hibernate)            | `@CreatedDate` (Spring Data)                          |
| --------------------- | ------------------------------------------- | ----------------------------------------------------- |
| Mechanism             | Hibernate value generator, applied at flush | JPA entity listener (`@PrePersist` / `@PreUpdate`)    |
| Setup                 | none, the annotation is enough              | `@EnableJpaAuditing` + `@EntityListeners`             |
| Who made the change   | no                                          | yes, `@CreatedBy` / `@LastModifiedBy` + `AuditorAware` |
| Database time         | yes (`source = SourceType.DB`)              | no, application time                                  |
| Time controllable in tests | no, JVM clock                          | yes, `DateTimeProvider`                               |
| Other stores          | no, JPA / Hibernate only                    | yes (MongoDB, JDBC...)                                |

Same limit for both: they only work when Hibernate writes the entity (not bulk JPQL updates, native SQL or JDBC), and the modification date only changes when the entity actually changed.
