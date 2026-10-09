# Cypher Query Language: End-to-End Tutorial

A hands-on guide to Neo4j's Cypher query language: creating nodes, defining data quality constraints, building relationships, and populating a graph with real data. Worked example uses a small company graph (Employees, Departments, Skills, Projects).

Tested against Cypher for Neo4j 5.x.

---

## 1. Core Concepts

| Concept | Meaning | Analogy |
|---|---|---|
| Node | An entity | A row in a table |
| Label | A category on a node | A table name |
| Property | A key-value attribute on a node/relationship | A column value |
| Relationship | A directed, typed connection between two nodes | A foreign key, but first-class |
| Pattern | ASCII-art syntax describing nodes and relationships | `(a)-[:REL]->(b)` |

Basic syntax shapes:

```cypher
()              // an anonymous node
(n)             // a node bound to variable n
(n:Person)      // a node with label Person
(n:Person {name: 'Ada'})   // a node with label and inline properties
()-[:KNOWS]->() // a directed relationship of type KNOWS
(a)-[r:KNOWS]->(b)         // relationship bound to variable r
```

---

## 2. Setting Up: Constraints Before Data

Define data quality constraints **before** loading data. In Neo4j, constraints are declared per label (or relationship type) and property.

### 2.1 Uniqueness constraints

Prevents duplicate values for a property across all nodes with a label. This also creates a backing index.

```cypher
CREATE CONSTRAINT employee_id_unique IF NOT EXISTS
FOR (e:Employee)
REQUIRE e.employeeId IS UNIQUE;

CREATE CONSTRAINT department_name_unique IF NOT EXISTS
FOR (d:Department)
REQUIRE d.name IS UNIQUE;

CREATE CONSTRAINT skill_name_unique IF NOT EXISTS
FOR (s:Skill)
REQUIRE s.name IS UNIQUE;
```

### 2.2 Existence constraints (property required)

Requires a property to be present whenever the label is applied. (Enterprise Edition feature; Community Edition supports uniqueness and node key only.)

```cypher
CREATE CONSTRAINT employee_email_exists IF NOT EXISTS
FOR (e:Employee)
REQUIRE e.email IS NOT NULL;

CREATE CONSTRAINT employee_hired_date_exists IF NOT EXISTS
FOR (e:Employee)
REQUIRE e.hiredOn IS NOT NULL;
```

### 2.3 Node key constraints (composite uniqueness + existence)

Enforces that a combination of properties is both present and unique — the graph equivalent of a composite primary key.

```cypher
CREATE CONSTRAINT employee_key IF NOT EXISTS
FOR (e:Employee)
REQUIRE (e.employeeId) IS NODE KEY;
```

### 2.4 Property type constraints

Enforces a property's data type (Neo4j 5.9+).

```cypher
CREATE CONSTRAINT employee_salary_type IF NOT EXISTS
FOR (e:Employee)
REQUIRE e.salary IS :: FLOAT;
```

### 2.5 Relationship property constraints

Constraints can also apply to relationship properties.

```cypher
CREATE CONSTRAINT works_on_allocation_exists IF NOT EXISTS
FOR ()-[r:WORKS_ON]-()
REQUIRE r.allocationPercent IS NOT NULL;
```

### 2.6 Inspecting and dropping constraints

```cypher
SHOW CONSTRAINTS;

DROP CONSTRAINT employee_id_unique IF EXISTS;
```

> Constraints throw an error and abort the write if violated — they don't silently skip bad rows. For bulk loads, catch violations early with `MERGE` (idempotent) rather than `CREATE` (always inserts, fails on duplicates).

---

## 3. Creating Nodes

### 3.1 Single node with properties

```cypher
CREATE (e:Employee {
  employeeId: 'E001',
  name: 'Ada Lovelace',
  email: 'ada@example.com',
  title: 'Principal Engineer',
  salary: 165000.0,
  hiredOn: date('2019-03-04')
});
```

### 3.2 Multiple labels on one node

A node can carry more than one label — useful for role overlap.

```cypher
CREATE (e:Employee:Manager {
  employeeId: 'E002',
  name: 'Grace Hopper',
  email: 'grace@example.com',
  title: 'VP Engineering',
  salary: 210000.0,
  hiredOn: date('2015-06-12')
});
```

### 3.3 Multiple nodes in one statement

```cypher
CREATE
  (d1:Department {name: 'Platform', budget: 4200000}),
  (d2:Department {name: 'Data', budget: 3100000}),
  (d3:Department {name: 'Growth', budget: 1800000});
```

### 3.4 CREATE vs MERGE

`CREATE` always inserts a new node — running the same statement twice creates duplicates. `MERGE` matches an existing node first and only creates one if no match is found. Use `MERGE` for anything driven by an external key, to keep loads idempotent.

```cypher
// Safe to re-run — will not duplicate Ada
MERGE (e:Employee {employeeId: 'E001'})
ON CREATE SET
  e.name = 'Ada Lovelace',
  e.email = 'ada@example.com',
  e.title = 'Principal Engineer',
  e.salary = 165000.0,
  e.hiredOn = date('2019-03-04')
ON MATCH SET
  e.lastSeen = datetime();
```

`ON CREATE SET` runs only when a new node is inserted; `ON MATCH SET` runs only when an existing node is found. This pattern is the standard way to make loads re-runnable without violating uniqueness constraints.

---

## 4. Creating Relationships (Edges)

Relationships are always directed and always have exactly one type. Both endpoint nodes must already be matched or created in the same statement.

### 4.1 Basic relationship creation

```cypher
MATCH (e:Employee {employeeId: 'E001'})
MATCH (d:Department {name: 'Platform'})
CREATE (e)-[:MEMBER_OF]->(d);
```

### 4.2 Relationship with properties

```cypher
MATCH (e:Employee {employeeId: 'E002'})
MATCH (d:Department {name: 'Data'})
CREATE (e)-[:MANAGES {since: date('2021-01-10')}]->(d);
```

### 4.3 Creating nodes and relationships together

```cypher
CREATE (e:Employee {employeeId: 'E003', name: 'Alan Turing', email: 'alan@example.com', title: 'Research Scientist', salary: 172000.0, hiredOn: date('2020-09-01')})
-[:MEMBER_OF]->
(d:Department {name: 'Data'});
```

Careful: if `Department {name: 'Data'}` already exists, this pattern creates a **duplicate** Department node, because `CREATE` doesn't check for existing matches. Prefer `MERGE` for the department side:

```cypher
MATCH (d:Department {name: 'Data'})
CREATE (e:Employee {employeeId: 'E004', name: 'Katherine Johnson', email: 'katherine@example.com', title: 'Data Scientist', salary: 158000.0, hiredOn: date('2022-02-14')})
CREATE (e)-[:MEMBER_OF]->(d);
```

### 4.4 Idempotent relationship creation with MERGE

```cypher
MATCH (e:Employee {employeeId: 'E001'})
MATCH (s:Skill {name: 'Graph Theory'})
MERGE (e)-[:HAS_SKILL]->(s);
```

`MERGE` on a relationship checks both endpoints and the relationship type (and properties, if specified in the pattern) before deciding whether to create it.

### 4.5 Relationship direction and multiple types

```cypher
MATCH (a:Employee {employeeId: 'E001'}), (b:Employee {employeeId: 'E003'})
CREATE (a)-[:MENTORS {startedOn: date('2023-01-15')}]->(b);

// Querying ignores direction with a plain dash
MATCH (a:Employee)-[:MENTORS]-(b:Employee) RETURN a.name, b.name;
```

---

## 5. Populating Data at Scale

### 5.1 UNWIND for batch inserts from a list

`UNWIND` expands a list into rows, letting you drive `MERGE`/`CREATE` from a parameter — this is the standard pattern for loading data from an application.

```cypher
UNWIND [
  {employeeId: 'E005', name: 'Radia Perlman', email: 'radia@example.com', title: 'Network Architect', salary: 175000.0, hiredOn: date('2018-11-20'), dept: 'Platform'},
  {employeeId: 'E006', name: 'Margaret Hamilton', email: 'margaret@example.com', title: 'Software Lead', salary: 180000.0, hiredOn: date('2017-05-09'), dept: 'Platform'},
  {employeeId: 'E007', name: 'Barbara Liskov', email: 'barbara@example.com', title: 'Distinguished Engineer', salary: 195000.0, hiredOn: date('2016-08-30'), dept: 'Data'}
] AS row
MERGE (e:Employee {employeeId: row.employeeId})
ON CREATE SET
  e.name = row.name,
  e.email = row.email,
  e.title = row.title,
  e.salary = row.salary,
  e.hiredOn = row.hiredOn
WITH e, row
MATCH (d:Department {name: row.dept})
MERGE (e)-[:MEMBER_OF]->(d);
```

### 5.2 Loading from CSV

Place CSV files in Neo4j's `import` directory (or reference an accessible URL), then load with `LOAD CSV`.

**employees.csv**
```csv
employeeId,name,email,title,salary,hiredOn,dept
E008,Hedy Lamarr,hedy@example.com,Systems Engineer,168000.0,2019-07-22,Growth
E009,Dorothy Vaughan,dorothy@example.com,Engineering Manager,190000.0,2015-03-18,Growth
```

```cypher
LOAD CSV WITH HEADERS FROM 'file:///employees.csv' AS row
MERGE (e:Employee {employeeId: row.employeeId})
ON CREATE SET
  e.name = row.name,
  e.email = row.email,
  e.title = row.title,
  e.salary = toFloat(row.salary),
  e.hiredOn = date(row.hiredOn)
WITH e, row
MATCH (d:Department {name: row.dept})
MERGE (e)-[:MEMBER_OF]->(d);
```

Notes:
- CSV values are always strings — cast explicitly with `toFloat()`, `toInteger()`, `date()`, `datetime()`.
- For files larger than a few hundred thousand rows, wrap the load in `CALL { ... } IN TRANSACTIONS OF 1000 ROWS` (Neo4j 4.4+/5.x) to avoid one giant transaction exhausting memory:

```cypher
LOAD CSV WITH HEADERS FROM 'file:///employees_large.csv' AS row
CALL {
  WITH row
  MERGE (e:Employee {employeeId: row.employeeId})
  ON CREATE SET e.name = row.name, e.email = row.email
} IN TRANSACTIONS OF 1000 ROWS;
```

### 5.3 Loading relationships from CSV

**works_on.csv**
```csv
employeeId,projectName,allocationPercent
E001,Graph Migration,60
E003,Graph Migration,40
E005,Network Redesign,100
```

```cypher
LOAD CSV WITH HEADERS FROM 'file:///works_on.csv' AS row
MATCH (e:Employee {employeeId: row.employeeId})
MERGE (p:Project {name: row.projectName})
MERGE (e)-[r:WORKS_ON]->(p)
SET r.allocationPercent = toInteger(row.allocationPercent);
```

### 5.4 Generating synthetic/bulk data with ranges

Useful for testing or seeding demo data at volume.

```cypher
UNWIND range(1, 100) AS i
CREATE (s:Skill {
  name: 'Skill-' + toString(i),
  category: CASE i % 3
    WHEN 0 THEN 'Engineering'
    WHEN 1 THEN 'Data'
    ELSE 'Design'
  END
});
```

---

## 6. Putting It Together: Full Worked Example

```cypher
// --- 1. Constraints (run once) ---
CREATE CONSTRAINT employee_id_unique IF NOT EXISTS
FOR (e:Employee) REQUIRE e.employeeId IS UNIQUE;

CREATE CONSTRAINT department_name_unique IF NOT EXISTS
FOR (d:Department) REQUIRE d.name IS UNIQUE;

CREATE CONSTRAINT project_name_unique IF NOT EXISTS
FOR (p:Project) REQUIRE p.name IS UNIQUE;

CREATE CONSTRAINT skill_name_unique IF NOT EXISTS
FOR (s:Skill) REQUIRE s.name IS UNIQUE;

// --- 2. Seed departments ---
UNWIND ['Platform', 'Data', 'Growth'] AS deptName
MERGE (:Department {name: deptName});

// --- 3. Seed employees + membership ---
UNWIND [
  {id: 'E001', name: 'Ada Lovelace', dept: 'Platform', title: 'Principal Engineer'},
  {id: 'E002', name: 'Grace Hopper', dept: 'Data', title: 'VP Engineering'},
  {id: 'E003', name: 'Alan Turing', dept: 'Data', title: 'Research Scientist'}
] AS row
MERGE (e:Employee {employeeId: row.id})
ON CREATE SET e.name = row.name, e.title = row.title
WITH e, row
MATCH (d:Department {name: row.dept})
MERGE (e)-[:MEMBER_OF]->(d);

// --- 4. Seed skills + assignment ---
UNWIND ['Graph Theory', 'Distributed Systems', 'Cryptography'] AS skillName
MERGE (:Skill {name: skillName});

MATCH (e:Employee {employeeId: 'E001'}), (s:Skill {name: 'Graph Theory'})
MERGE (e)-[:HAS_SKILL {level: 'Expert'}]->(s);

MATCH (e:Employee {employeeId: 'E003'}), (s:Skill {name: 'Cryptography'})
MERGE (e)-[:HAS_SKILL {level: 'Expert'}]->(s);

// --- 5. Manager relationship ---
MATCH (mgr:Employee {employeeId: 'E002'}), (report:Employee {employeeId: 'E003'})
MERGE (mgr)-[:MANAGES]->(report);
```

---

## 7. Querying What You Built

```cypher
// All employees and their department
MATCH (e:Employee)-[:MEMBER_OF]->(d:Department)
RETURN e.name AS employee, d.name AS department
ORDER BY department;

// Who manages whom
MATCH (mgr:Employee)-[:MANAGES]->(report:Employee)
RETURN mgr.name AS manager, report.name AS report;

// Employees with a given skill
MATCH (e:Employee)-[r:HAS_SKILL]->(s:Skill {name: 'Graph Theory'})
RETURN e.name, r.level;

// Two-hop: colleagues in the same department
MATCH (a:Employee)-[:MEMBER_OF]->(d:Department)<-[:MEMBER_OF]-(b:Employee)
WHERE a.employeeId < b.employeeId
RETURN a.name, b.name, d.name AS department;

// Shortest path between two employees through any relationship
MATCH p = shortestPath((a:Employee {employeeId: 'E001'})-[*..5]-(b:Employee {employeeId: 'E003'}))
RETURN p;
```

---

## 8. Updating and Deleting

```cypher
// Update a property
MATCH (e:Employee {employeeId: 'E001'})
SET e.title = 'Distinguished Engineer';

// Add a label
MATCH (e:Employee {employeeId: 'E001'})
SET e:Mentor;

// Remove a property
MATCH (e:Employee {employeeId: 'E001'})
REMOVE e.title;

// Delete a relationship
MATCH (:Employee {employeeId: 'E001'})-[r:MENTORS]->(:Employee {employeeId: 'E003'})
DELETE r;

// Delete a node and all its relationships
MATCH (e:Employee {employeeId: 'E999'})
DETACH DELETE e;
```

`DELETE` on a node with existing relationships fails unless you use `DETACH DELETE`, which removes the node's relationships first.

---

## 9. Common Pitfalls

Using `CREATE` where `MERGE` was needed produces duplicate nodes on re-run, silently corrupting the graph — always use `MERGE` for anything keyed by an external identifier.

Forgetting `WITH` between a `MERGE`/`SET` clause and a subsequent `MATCH` can cause the query to lose the binding or produce a cartesian product; chain clauses deliberately with `WITH e, row` when carrying variables forward.

Constraints only catch violations at write time — retrofitting a constraint onto data that already contains duplicates will fail with a clear error listing the conflict, so clean up first with a `MATCH ... WITH ... WHERE count > 1` dedup query.

`LOAD CSV` without `IN TRANSACTIONS` on large files runs as one transaction and can exhaust heap memory — always batch large loads.

---

## 10. Reference: Constraint Types Summary

| Constraint type | Guarantees | Edition |
|---|---|---|
| `IS UNIQUE` | No two nodes/relationships share the property value | Community + Enterprise |
| `IS NODE KEY` | Property (or set of properties) is present and unique | Enterprise only |
| `IS NOT NULL` | Property must be present | Enterprise only |
| `IS :: <TYPE>` | Property must match the declared type | Enterprise only, Neo4j 5.9+ |
| `IS RELATIONSHIP KEY` | Relationship-level equivalent of node key | Enterprise only |

Community Edition supports property uniqueness constraints; existence, node key, and type constraints require Enterprise Edition.
