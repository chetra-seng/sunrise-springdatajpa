---
theme: seriph
background: https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1920&h=1080&fit=crop
title: Spring Data JPA — Sunrise Java Course
class: text-center
transition: slide-left
mdc: true
favicon: https://spring.io/favicon.svg
controls: true
download: true
info: |
  ## Spring Data JPA — Sunrise Java Course

  A comprehensive introduction to Spring Data JPA — from relational database
  fundamentals and SQL to entity mapping, repository abstraction, and modeling
  real-world data relationships in a Spring Boot application.

keywords: Spring Data JPA, Java, PostgreSQL, Hibernate, JPA, Spring Boot, Entities, Relationships
author: Sunrise Java Course
description: A comprehensive introduction to Spring Data JPA — from relational database fundamentals and SQL to entity mapping, repository abstraction, and modeling real-world data relationships in a Spring Boot application.
themeConfig:
  primary: '#6db33f'

seoMeta:
  ogType: website
  ogTitle: Spring Data JPA — Sunrise Java Course
  ogDescription: A comprehensive introduction to Spring Data JPA — entity mapping, repository abstraction, and real-world data relationships in Spring Boot.
  ogImage: https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&h=630&fit=crop

htmlAttrs:
  lang: en
  dir: ltr
---

# Spring Data JPA

Sunrise Java Course

<div class="pt-12">
  <span class="px-2 py-1">
    From in-memory to production — powered by Spring Data JPA
  </span>
</div>

---

# What We'll Learn

- 🗄️ **SQL** — how databases actually store and query data
- ☕ **Spring Data JPA** — let Java generate the SQL for you
- 🔗 **Relationships** — link tables together with `@OneToMany` and `@ManyToMany`
- 🚀 **Migration** — take your existing Task API and connect it to PostgreSQL

---
src: ./sections/01-database-fundamentals.md
---

---
src: ./sections/02-postgresql-setup.md
---

---
src: ./sections/03-data-types-tables.md
---

---
src: ./sections/04-crud-operations.md
---

---
src: ./sections/05-joins.md
---

---
src: ./sections/06-aggregations-subqueries.md
---

---
src: ./sections/07-indexes-transactions.md
---

---
src: ./sections/08-spring-jpa-intro.md
---

---
src: ./sections/09-entity-mapping.md
---

---
src: ./sections/10-jpa-relationships.md
---

---
src: ./sections/11-repositories.md
---

---
src: ./sections/12-query-methods.md
---

---
src: ./sections/19-task-migration-lab.md
---

---
src: ./sections/17-custom-queries.md
---

---
src: ./sections/18-specifications.md
---

---
src: ./sections/13-pagination-sorting.md
---

---
src: ./sections/14-transactions-performance.md
---

---
src: ./sections/20-relationships.md
---

---
src: ./sections/15-rest-apis.md
---

---
src: ./sections/16-testing-best-practices.md
---

---
layout: center
class: text-center
---

# Course Complete!

You've migrated a real Task Management API from in-memory storage to PostgreSQL using Spring Data JPA.

<div class="text-left pt-8">

**What you changed:**
- `Task.java` → added `@Entity`, `@Id`, `@GeneratedValue`, `@CreationTimestamp`
- `TaskRepository.java` → replaced `ConcurrentHashMap` class with `JpaRepository` interface
- `pom.xml` → added `spring-boot-starter-data-jpa` + `postgresql`
- `application.properties` → added datasource + JPA config

**What stayed the same:** `TaskServiceImpl`, `TaskController`, `TaskMapper` — untouched

**Resources:**
- Spring Data JPA Docs: <a href="https://docs.spring.io/spring-data/jpa/reference/" target="_blank">docs.spring.io/spring-data/jpa</a>
- Hibernate Docs: <a href="https://hibernate.org/orm/documentation/" target="_blank">hibernate.org</a>
- PostgreSQL Docs: <a href="https://www.postgresql.org/docs/" target="_blank">postgresql.org/docs</a>

</div>
