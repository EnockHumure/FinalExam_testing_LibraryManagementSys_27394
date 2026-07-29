## (AYA MAKURU ARAKENEWE CYANE NKUMU DEVELOPER)

# AUCA Library Management System — Final Project (Summer Semester 2025)

> Course: Software Testing and Techniques
> Stack: Java 21 · Maven · Hibernate ORM 6.5 · PostgreSQL · JUnit 4
> Database: `auca_library_db`
> Package root: `com.auca.library`

---
to day mostly we focus on setting up fings like relation ships, the package we will need and other related thing and  setting up pom.xml 
so it will be easly to as
for analyses and work on project and database and tables we will need there 

## Project Overview

This system manages physical (hard-copy) books in the AUCA library. It tracks who borrows books, how many are borrowed, return deadlines, and applies late-return charges. Users self-register and choose a membership plan. The librarian manages the physical space (rooms, shelves, books).
basically to day we are planning how the  we will integrate and design every thing that we should use in our system  this is then we will 
continue with the codes here is the plan of how we wil impliment the system

---


## Project Structure

```
src/main/java/com/auca/library/
├── domain/        ← Entities (JPA/Hibernate mapped classes)
├── dao/           ← Data Access Objects (Hibernate Session + HQL)
├── service/       ← Business rules & logic
// this file will be implimented because of borrow limit ├── exception/     ← Custom exceptions (BorrowLimitExceededException)
└── util/          ← HibernateUtil (SessionFactory singleton)
```

---

## Database Schema (EERD)

### Tables & Columns

```
┌─────────────────────────────────────────────────────────────────────┐
│  locations                                                          │
│  ─────────────────────────────────────────────────────────────────  │
│  id              UUID  PK                                           │
│  location_code   VARCHAR  UNIQUE NOT NULL                           │
│  location_name   VARCHAR  NOT NULL                                  │
│  location_type   ENUM(PROVINCE, DISTRICT, SECTOR, CELL, VILLAGE)   │
│  parent_id       UUID  FK → locations.id  (self-ref, nullable)      │
└─────────────────────────────────────────────────────────────────────┘
         │ (self-referencing ManyToOne)
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  users  (extends Person — @MappedSuperclass, no own table)          │
│  ─────────────────────────────────────────────────────────────────  │
│  id              UUID  PK                                           │
│  first_name      VARCHAR                                            │
│  last_name       VARCHAR                                            │
│  phone_number    VARCHAR                                            │
│  gender          VARCHAR                                            │
│  location_id     UUID  FK → locations.id                            │
│  username        VARCHAR  UNIQUE NOT NULL                           │
│  password        VARCHAR  NOT NULL                                  │
│  role            VARCHAR  (LIBRARIAN | READER)                      │
└─────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  membership_types                                                   │
│  ─────────────────────────────────────────────────────────────────  │
│  id              UUID  PK                                           │
│  name            VARCHAR  UNIQUE NOT NULL  (Gold/Silver/Striver)    │
│  price_per_day   INT  NOT NULL                                      │
│  max_books       INT  NOT NULL                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  memberships                                                        │
│  ─────────────────────────────────────────────────────────────────  │
│  id                  UUID  PK                                       │
│  user_id             UUID  FK → users.id                            │
│  membership_type_id  UUID  FK → membership_types.id                 │
│  status              ENUM(ACTIVE, PENDING, EXPIRED)                 │
│  registration_date   DATE                                           │
│  expiring_date       DATE                                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  rooms                                                              │
│  ─────────────────────────────────────────────────────────────────  │
│  id          UUID  PK                                               │
│  room_code   VARCHAR  UNIQUE NOT NULL                               │
│  room_name   VARCHAR                                                │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  shelves                                                            │
│  ─────────────────────────────────────────────────────────────────  │
│  id               UUID  PK                                          │
│  shelf_code       VARCHAR  NOT NULL                                 │
│  available_stock  INT                                               │
│  room_id          UUID  FK → rooms.id                               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  books                                                              │
│  ─────────────────────────────────────────────────────────────────  │
│  id                UUID  PK                                         │
│  title             VARCHAR  NOT NULL                                │
│  isbn              VARCHAR                                          │
│  publisher         VARCHAR                                          │
│  publication_year  INT                                              │
│  status            ENUM(AVAILABLE, BORROWED)                        │
│  shelf_id          UUID  FK → shelves.id                            │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  borrowers                                                          │
│  ─────────────────────────────────────────────────────────────────  │
│  id           UUID  PK                                              │
│  reader_id    UUID  FK → users.id                                   │
│  book_id      UUID  FK → books.id                                   │
│  pickup_date  DATE  NOT NULL                                        │
│  due_date     DATE  NOT NULL                                        │
│  return_date  DATE  (null = not yet returned)                       │
│  fine         INT   DEFAULT 0                                       │
│  late_fee     INT   DEFAULT 0                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## EERD — Entity Relationship Diagram

```
                    ┌──────────────┐
                    │   Location   │◄──────────────────┐
                    │──────────────│  self-ref          │
                    │ id (PK)      │  ManyToOne         │
                    │ location_code│  (parent_id)       │
                    │ location_name│                    │
                    │ location_type│────────────────────┘
                    │ parent_id(FK)│
                    └──────┬───────┘
                           │ ManyToOne (1 location → many users)
                           ▼
                    ┌──────────────┐
                    │     User     │
                    │──────────────│
                    │ id (PK)      │
                    │ first_name   │
                    │ last_name    │
                    │ phone_number │
                    │ gender       │
                    │ location_id  │
                    │ username     │
                    │ password     │
                    │ role         │
                    └──────┬───────┘
              ┌────────────┴────────────┐
              │ ManyToOne               │ ManyToOne
              ▼                         ▼
   ┌──────────────────┐       ┌──────────────────┐
   │   Membership     │       │    Borrower       │
   │──────────────────│       │──────────────────│
   │ id (PK)          │       │ id (PK)          │
   │ user_id (FK)     │       │ reader_id (FK)   │
   │ type_id (FK)     │       │ book_id (FK)     │
   │ status           │       │ pickup_date      │
   │ registration_date│       │ due_date         │
   │ expiring_date    │       │ return_date      │
   └────────┬─────────┘       │ fine             │
            │ ManyToOne       │ late_fee         │
            ▼                 └────────┬─────────┘
   ┌──────────────────┐                │ ManyToOne
   │  MembershipType  │                ▼
   │──────────────────│       ┌──────────────────┐
   │ id (PK)          │       │      Book        │
   │ name             │       │──────────────────│
   │ price_per_day    │       │ id (PK)          │
   │ max_books        │       │ title            │
   └──────────────────┘       │ isbn             │
                              │ publisher        │
                              │ publication_year │
                              │ status           │
                              │ shelf_id (FK)    │
                              └────────┬─────────┘
                                       │ ManyToOne
                                       ▼
                              ┌──────────────────┐
                              │      Shelf       │
                              │──────────────────│
                              │ id (PK)          │
                              │ shelf_code       │
                              │ available_stock  │
                              │ room_id (FK)     │
                              └────────┬─────────┘
                                       │ ManyToOne
                                       ▼
                              ┌──────────────────┐
                              │      Room        │
                              │──────────────────│
                              │ id (PK)          │
                              │ room_code        │
                              │ room_name        │
                              └──────────────────┘
```

### Relationship Summary

| Entity | Related To | Type | FK Column |
|--------|-----------|------|-----------|
| Location | Location (parent) | ManyToOne (self) | parent_id |
| User | Location | ManyToOne | location_id |
| Membership | User | ManyToOne | user_id |
| Membership | MembershipType | ManyToOne | membership_type_id |
| Borrower | User | ManyToOne | reader_id |
| Borrower | Book | ManyToOne | book_id |
| Book | Shelf | ManyToOne | shelf_id |
| Shelf | Room | ManyToOne | room_id |

---

## Membership Plans

| Plan | Price/Day | Max Books |
|------|-----------|-----------|
| Gold | 50 Rwf | 5 |
| Silver | 30 Rwf | 3 |
| Striver | 10 Rwf | 2 |

-
