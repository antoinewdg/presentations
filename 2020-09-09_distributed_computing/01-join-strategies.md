# Join strategies

---

## Introduction

- how to make a simple join between two tables if Postgres did not do it for us ?
- pre-requisite to understanding how to do it in a distributed fashion

---

## Context

Two SQL tables

```sql
CREATE TABLE company (
  id integer PRIMARY KEY,
  name text
);
```

```sql
CREATE TABLE employee (
  id integer PRIMARY KEY,
  company_id integer NOT NULL REFERENCES company(id),
  name text
);
```

- `M` companies
- `N` employees

---

## Context


How do you perform the following ?

```sql
SELECT
  employee.id,
  employee.name,
  company.name as company_name
FROM employee
INNER JOIN company ON company.id = employee.company_id
```


**Note:** this is a very simple case:

- an employee can only have one company
- it always exists

---

## Context

```python
@dataclass
class Company:
    id: int
    name: str

@dataclass
class Employee:
    id: int
    company_id: int
    name: str

@dataclass
class Result:
    id: int
    name: str
    company_name: str
```

---

## Objective

```python
def perform_join(
    employees: List[Employee], companies: List[Company]
) -> List[Result]:
    ...

```

---

## Objective

```python
def perform_join(
    employees: List[Employee], companies: List[Company]
) -> List[Result]:
    ...

```

Any ideas ?

---

## Nested loop

```python
def perform_nested_loop_join(
    employees: List[Employee], companies: List[Company]
) -> List[Result]:
    results = []

    for employee in employees:
        for company in companies:
            if employee.company_id == company.id:
                results.append(make_result(employee, company))

    return results
```

---

## Nested loop

Complexity: `O(M * N)`

Pros:

- super simple
- low RAM usage

Cons:

- very slow if inputs are big

---

## Hash join

```python
def perform_hash_join(
    employees: List[Employee], companies: List[Company]
) -> List[Result]:
    result = []

    # Create a hash table that maps a company id to the company
    company_by_id = {c.id: c for c in companies}

    # Loop over employees
    for employee in employees:
        company = company_by_id[employee.company_id]
        result.append(make_result(employee, company))

    return result
```

---

## Hash join

Complexity:

- `O(M)` to compute `company_by_id`
- `O(N)` for the loop

Pros:

- the fastest available
- `company_by_id` can be pre-computed (index)

Cons:

- high RAM usage (hash tables are BIG and can't be spilled to disk)

---


## Sort-merge join

```python
def perform_sort_merge_join(
    employees: List[Employee], companies: List[Company]
) -> List[Result]:
    # First: sort the inputs by join key
    employees = sorted(employees, key=lambda e: e.company_id)
    companies = sorted(companies, key=lambda c: c.id)

    # Then loop over both simultaneously
    result = []
    company_idx = 0

    for employee in employees:
        while companies[company_idx].id != employee.company_id:
            company_idx += 1
        result.append(make_result(employee, companies[company_idx]))

    return result

```

---

## Sort-merge join

```python
employees = [
    Employee(company_id=1, id=1, name="Employee A"),
    Employee(company_id=2, id=5, name="Employee E"),
    Employee(company_id=4, id=4, name="Employee D"),
    Employee(company_id=5, id=2, name="Employee B"),
    Employee(company_id=5, id=3, name="Employee C"),
]

companies = [
    Company(id=1, name="Company A"),
    Company(id=2, name="Company B"),
    Company(id=3, name="Company C"),
    Company(id=4, name="Company D"),
    Company(id=5, name="Company E"),
    Company(id=6, name="Company F"),
]

```

---

## Sort-merge join

Complexity `O(M*logM + N*logN)`:

- `O(M*logM + N*logN)` for sorting phase
- `O(M + N)` for the loop

Pros:

- fast
- low RAM usage (depends on the sorting algorithm)
==> the most balanced

Cons:

- not as fast as hash join

---

## Aside: BTrees

In Postgres, most of your indexes are not hash tables, they are BTrees, a data structure that

- has fast lookup time (`O(logN)`, but really small)
- keeps values in a sorted order
- can be spilled to disk

You can use variants of hash and sort-merge joins with BTrees.

---

# Questions ?

---

## Additional resources

- [SQL performance explained](https://mytraffic.atlassian.net/wiki/spaces/TECH/pages/31064199/Livres+utiles) for information about join types (and a lot of other intersting stuff)
- [Use the index Luke](https://use-the-index-luke.com/sql/anatomy/the-tree) for an in-depth look at BTrees
