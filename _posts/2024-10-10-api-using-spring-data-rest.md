---
title: How to make an API using Spring Data REST
date: 2024-10-10 11:30:0 -0300
categories: [Web, API]
tags: [java, spring, api, rest]
comments: true
---

## What is Spring Data REST?

[Spring Data REST](https://docs.spring.io/spring-data/rest/reference/index.html) is a module of the Spring framework that automatically exposes RESTful APIs for your Spring Data repositories. It simplifies the process of building data-driven services by converting your database entities into HTTP resources, handling common operations like CRUD, pagination, and sorting without the need to write boilerplate code. By using conventions, it automatically maps your repository methods to HTTP endpoints, allowing you to focus on your application's logic while leveraging powerful features like HATEOAS links and data projections.

## Stack used in the project

We are going to use the following technologies for building our API:

- Java 21
- Spring Boot 3.3.4
- Spring Data REST
- PostgreSQL
- Docker

## API Requirements

The project we're going to make is an API for adopting animals. We need to ensure the following requirements:

1. An endpoint for creating a new animal, that receives the following data:
: - Name
- Description
- Image URL
- Category
- Birth Date
- Status (AVAILABLE, ADOPTED)

2. An endpoint for getting animals, that returns:
: - ID
- Name
- Description
- Image URL
- Category
- Birth Date
- Age <- _Notice the extra field_
- Status (AVAILABLE, ADOPTED)

3. **An endpoint to change an animal status**

4. **Swagger documentation for all endpoints**

## Project Setup

### Gradle

Add the following dependencies to your `build.gradle`{: .filepath}:

```gradle
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-data-rest'
	implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
	implementation 'org.springdoc:springdoc-openapi-data-rest:1.8.0'
	compileOnly 'org.projectlombok:lombok'
	developmentOnly 'org.springframework.boot:spring-boot-devtools'
	runtimeOnly 'org.postgresql:postgresql'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```
{: file="build.gradle" }

### Application properties

The application properties for our project:

```yml
spring:
  application:
    name: adoption
  devtools:
    restart:
      enabled: true
      exclude: static/**,public/**
    livereload:
      enabled: true
  datasource:
    url: jdbc:postgresql://localhost:5432/adoption
    username: postgres
    password: root
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    hibernate:
      ddl-auto: update
  data:
    rest:
      basePath: /api

springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html

```
{: file="resources/application.yml" }

### Postgres

Heres our *compose.yml* with PostgreSQL docker configuration
```yml
services:
  postgres:
    image: 'postgres:latest'
    ports:
      - '5432:5432'
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: root
      POSTGRES_DB: adoption
```
{: file="compose.yml" }

## Model

Explaining some annotations:

1. @EntityListeners(AuditingEntityListener.class) -> This gives us the metadata of when an object was created or updated.

2. @JsonProperty(access = JsonProperty.Access.READ_ONLY) -> This removes the field from the POST endpoint that Spring Data REST creates.

```java
@EntityListeners(AuditingEntityListener.class)
@Entity
@Table(name = "animals")
@Data
public class Animal {
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    @Column(name = "animal_id")
    private Long id;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @CreatedDate
    @Column(name = "created_date", nullable = false, updatable = false)
    private Date createdDate;

    @JsonProperty(access = JsonProperty.Access.READ_ONLY)
    @LastModifiedDate
    @Column(name = "last_modified_date")
    private Date lastModifiedDate;

    @Column(name = "name", length = 255, nullable = false)
    private String name;

    @Column(name = "description", length = 255, nullable = true)
    private String description;

    @Column(name = "image_url", length = 255, nullable = true)
    private String imageURL;

    @Column(name = "category", length = 255, nullable = false)
    private String category;

    @Column(name = "birth_date", length = 255, nullable = false)
    private Date birthDate;

    @Column(name = "status", length = 255, nullable = true)
    private Status status;
    public enum Status {DISPONIVEL, ADOTADO}
}
```
{: file="domain/model/Animal.java" }

### Projection

We need a DTO for GET requests that have the animal age, so we're going to create a *projection* package inside the *model* package with our custom projection:

```java
@Projection(name = "animalWithAge", types = Animal.class)
public interface AnimalWithAge {
    Long getId();
    String getName();
    String getDescription();
    String getImageURL();
    String getCategory();
    LocalDate getBirthDate();

    default int getAge() {
        return Period.between(getBirthDate(), LocalDate.now()).getYears();
    }

    Animal.Status getStatus();
}
```
{: file="domain/model/projection/AnimalWithAge.java" }

## Repository

Spring Data REST automatically reads which interfaces our repository extends and configures the endpoint accordingly.

Since we're using the *JpaRepository*, it already extends the *PagingAndSortingRepository* and *CrudRepository*, making our API more complete.

The *Tag* annotation is to group our controller under the same name for swagger, for when we create our custom endpoint to changing the status.

```java
@RepositoryRestResource(collectionResourceRel = "animals", path = "animals")
@Tag(name = "Animal Controller", description = "Animal Management Endpoints")
public interface AnimalRepository extends JpaRepository<Animal, Long> {}
```
{: file="domain/repository/AnimalRepository.java" }

## Controller
