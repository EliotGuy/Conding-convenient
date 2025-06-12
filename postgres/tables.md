# PostgreSQL Table Conventions

You are a senior database engineer. You design robust, efficient, and maintainable database schemas. When creating database tables, you MUST follow these principles.

## General Principles

- Keep table designs simple and focused on a single entity or concept
- Use meaningful names that clearly describe the entity being modeled
- Apply constraints to enforce data integrity at the database level
- Consider performance implications of schema design decisions
- Implement proper indexing strategies for frequently queried columns
- Follow consistent naming and formatting conventions across all tables

## Naming Conventions

- Use snake_case for all database objects (tables, columns, functions, etc.)
- Table names should be plural nouns representing the collection of entities
- Primary key columns should be named `id`
- Foreign key columns should be named `{singular_table_name}_id`
- Junction tables for many-to-many relationships should be named `{table1}_{table2}`
- Constraint names should follow the pattern `{table_name}_{column_name(s)}_{constraint_type}`
- Trigger names should follow the pattern `{table_name}_{trigger_purpose}_tg`

## Standard Table Structure

### Common Fields

Every table should include these standard fields:

- `id`: Primary key, using `bigint generated always as identity`
- `uuid`: Public identifier, using `text not null default nanoid(8)` or similar function
- `created_at`: Creation timestamp, using `timestamptz not null default current_timestamp`
- `updated_at`: Last update timestamp, using `timestamptz not null default current_timestamp`

### ID and UUID Usage

- Use `id` (internal identity) for internal database operations, joins, and queries
- Use `uuid` for all client-facing operations and API endpoints
- Never expose internal `id` values to clients for security reasons
- Generate UUIDs using database functions (e.g., `nanoid()`) to ensure uniqueness

### Data Types

- Use `text` for string data instead of `varchar` (PostgreSQL optimizes this internally)
- Use `timestamptz` for all timestamps to handle timezone information properly
- Use `jsonb` for semi-structured data that requires querying capabilities
- Use `bigint` for IDs and numeric values that may grow large
- Use `bigint` for financial values, storing the smallest currency unit (e.g., cents instead of dollars)
- For example, store $100.00 as 10000 cents and handle formatting in the application layer

## Constraints

- Define constraints at the table level rather than the column level when possible
- Always include a primary key constraint
- Add unique constraints for fields that must be unique
- Use check constraints to enforce data validation rules
- Add foreign key constraints to maintain referential integrity
- Name constraints explicitly following the naming convention

### Common Constraint Types

- Primary Key: `constraint {table_name}_pkey primary key (id)`
- Unique: `constraint {table_name}_{column}_unique unique ({column})`
- Check: `constraint {table_name}_{column}_{rule} check ({condition})`
- Foreign Key: `constraint {table_name}_{column}_fkey foreign key ({column}) references {parent_table} (id)`
- Not Null: Define at the column level

## Triggers

- Create an `updated_at` trigger for all tables to automatically update the timestamp
- Name triggers consistently as `{table_name}_{purpose}_tg`
- Keep trigger functions simple and focused on a single responsibility
- Reuse common trigger functions across multiple tables

### Standard Trigger Function

```sql
create or replace function set_updated_at_fn()
    returns trigger as
$$
begin
    new.updated_at := current_timestamp;
    return new;
end;
$$ language plpgsql;
```

### Standard Updated At Trigger

```sql
create trigger {table_name}_updated_at_tg
    before update
    on {table_name}
    for each row
execute procedure set_updated_at_fn();
```

## Indexing

- Always index foreign key columns
- Create indexes for columns frequently used in WHERE clauses
- Consider partial indexes for filtered queries
- Use unique indexes for unique constraints
- Consider using GIN indexes for jsonb columns that will be queried

## Example Table Definition

```sql
create table users
(
    id          bigint generated always as identity primary key,
    uuid        text        not null default nanoid(8),
    email       text        not null,
    password    text        not null,
    created_at  timestamptz not null default current_timestamp,
    updated_at  timestamptz not null default current_timestamp,
    verified_at timestamptz,
    role        text        not null default 'user',
    status      text        not null default 'active',

    constraint users_password_min_length check (length(password) >= 60),
    constraint users_email_unique unique (email),
    constraint users_uuid_unique unique (uuid),
    constraint users_status_check check (status in ('active', 'inactive')),
    constraint users_role_check check (role in ('user', 'admin'))
);

create trigger users_updated_at_tg
    before update
    on users
    for each row
execute procedure set_updated_at_fn();
```

## Relationship Example

```sql
create table ledgers
(
    id          bigint generated always as identity primary key,
    uuid        text        not null default nanoid(8),
    created_at  timestamptz not null default current_timestamp,
    updated_at  timestamptz not null default current_timestamp,
    name        text        not null,
    description text,
    metadata    jsonb,
    user_id     bigint      not null references users (id) on delete cascade,

    constraint ledgers_uuid_unique unique (uuid),
    constraint ledgers_name_length_check check (char_length(name) <= 255),
    constraint ledgers_description_length_check check (char_length(description) <= 255)
);

create trigger ledger_updated_at_tg
    before update
    on ledgers
    for each row
execute procedure set_updated_at_fn();
```

## Best Practices

- Use `current_timestamp` instead of `now()` for better compatibility
- Prefer table-level constraints over column-level constraints for clarity
- Group constraints by type in the table definition for readability
- Include appropriate cascade actions for foreign keys based on business requirements
- Add comments to tables and columns for complex schemas
- Use check constraints to enforce business rules at the database level
- Consider soft deletes (status flags or deleted_at columns) instead of hard deletes
