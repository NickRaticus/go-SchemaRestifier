# GenerateRepoLayer Flow Diagram

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      GenerateRepoLayer()                             │
│  Input: g.Schemas []parser.Schema                                    │
│  Output: Generated repository files with CRUD methods                │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────────┐
        │   FOR EACH schema IN g.Schemas          │
        └─────────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────────┐
        │   Phase 1: Field Classification         │
        │   (Iterate through *schema.Columns)     │
        └─────────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────────┐
        │   Phase 2: Build Repository Code        │
        │   (Generate CRUD methods)               │
        └─────────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────────┐
        │   Phase 3: Write Repository File        │
        │   (Output: {table}_repository.go)       │
        └─────────────────────────────────────────┘
```

---

## Phase 1: Field Classification (Per Schema)

```
┌────────────────────────────────────────────────────────────────────────┐
│                  FOR EACH column IN *schema.Columns                    │
└────────────────────────────────────────────────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────┐
        │         Check column.Capabilities                 │
        └───────────────────────────────────────────────────┘
                                    ↓
    ┌──────────────┬──────────────┬──────────────┬──────────────┐
    ↓              ↓              ↓              ↓              ↓
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ select? │  │ filter? │  │  sort?  │  │ search? │  │   PK?   │
└─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘
    │              │              │              │              │
    ↓              ↓              ↓              ↓              ↓
  append       append       append       append       store
selectFields filterFields sortFields searchFields primaryKeyField
    │              │              │              │              │
    └──────────────┴──────────────┴──────────────┴──────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────┐
        │         Check column.Operations                   │
        └───────────────────────────────────────────────────┘
                                    ↓
            ┌──────────────┬──────────────┐
            ↓              ↓              ↓
      ┌─────────┐    ┌─────────┐    ┌─────────┐
      │ create? │    │ update? │    │   get?  │
      └─────────┘    └─────────┘    └─────────┘
            │              │              │
            ↓              ↓              ↓
    Check autoSet?   Check !PK?      Used in
    If not auto:     If valid:       response
        │                  │              │
        ↓                  ↓              ↓
      append           append         (implicit
  createFields     updateFields      in SELECT)
  createParams     updateParams
  createValues     updateSets
        │                  │
        └──────────────────┴───────────────────────┐
                                                    ↓
                        ┌────────────────────────────────────┐
                        │   Result: 7 Field Collections      │
                        │   + 1 Primary Key Field            │
                        └────────────────────────────────────┘
```

### Field Collections Created

```
┌──────────────────┬────────────────────────────────────────────────┐
│ Collection       │ Purpose / SQL Context                          │
├──────────────────┼────────────────────────────────────────────────┤
│ selectFields     │ SELECT {fields} FROM table                     │
│ createFields     │ INSERT INTO table ({fields}) VALUES (...)      │
│ createParams     │ m.Field1, m.Field2 (for VALUES placeholders)  │
│ createValues     │ $1, $2, $3 (PostgreSQL placeholders)          │
│ updateFields     │ UPDATE table SET {field}=$1, ...               │
│ updateParams     │ m.Field1, m.Field2 (for UPDATE values)        │
│ updateSets       │ field1=$1, field2=$2 (SET clause)             │
│ filterFields     │ WHERE {field}=value (validation list)          │
│ sortFields       │ ORDER BY {field} (validation list)             │
│ searchFields     │ WHERE {field} ILIKE '%term%' OR ...            │
│ primaryKeyField  │ WHERE {pk}=$1, RETURNING {pk}                  │
└──────────────────┴────────────────────────────────────────────────┘
```

---

## Phase 2: Build Repository Code

```
┌────────────────────────────────────────────────────────────────────┐
│              Generate Repository Struct & Constructor              │
└────────────────────────────────────────────────────────────────────┘
                                ↓
    type UserProfileRepository struct { DB *sqlx.DB }
    func NewUserProfileRepository(db *sqlx.DB) *UserProfileRepository
                                ↓
┌────────────────────────────────────────────────────────────────────┐
│                     Generate CRUD Methods                          │
└────────────────────────────────────────────────────────────────────┘
                                ↓
        ┌───────────────────────────────────────┐
        │                                       │
        ↓                                       ↓
┌───────────────────┐              ┌───────────────────────┐
│  1. Create()      │              │   Uses:               │
│  ─────────────    │              │   - createFields      │
│  Input:           │              │   - createValues      │
│   ctx context     │              │   - createParams      │
│   m *model.Table  │              │   - primaryKeyField   │
│  Output:          │◄─────────────┤                       │
│   error           │              │   SQL Pattern:        │
│                   │              │   INSERT INTO table   │
│  Returns:         │              │   (f1, f2, f3)       │
│   - nil on OK     │              │   VALUES ($1,$2,$3)   │
│   - wrapped err   │              │   RETURNING pk        │
└───────────────────┘              └───────────────────────┘
        │
        ↓
┌───────────────────┐              ┌───────────────────────┐
│  2. GetByID()     │              │   Uses:               │
│  ─────────────    │              │   - selectFields      │
│  Input:           │              │   - primaryKeyField   │
│   ctx context     │              │                       │
│   id int64        │              │   SQL Pattern:        │
│  Output:          │◄─────────────┤   SELECT f1,f2,f3     │
│   *model.Table    │              │   FROM table          │
│   error           │              │   WHERE pk=$1         │
│                   │              │                       │
│  Returns:         │              │   Special Case:       │
│   - (nil, nil)    │              │   sql.ErrNoRows →     │
│     if not found  │              │   return (nil, nil)   │
│   - (model, nil)  │              │                       │
│   - (nil, err)    │              │                       │
└───────────────────┘              └───────────────────────┘
        │
        ↓
┌───────────────────┐              ┌───────────────────────┐
│  3. GetAll()      │              │   Uses:               │
│  ─────────────    │              │   - selectFields      │
│  Input:           │              │   - filterFields      │
│   ctx context     │              │   - sortFields        │
│   filters map     │              │                       │
│   sortBy string   │              │   SQL Pattern:        │
│   limit int       │◄─────────────┤   SELECT f1,f2,f3     │
│   offset int      │              │   FROM table          │
│  Output:          │              │   [WHERE f1=$1        │
│   []*model.Table  │              │    AND f2=$2]         │
│   error           │              │   [ORDER BY f3 DESC]  │
│                   │              │   LIMIT $x OFFSET $y  │
│  Returns:         │              │                       │
│   - (items, nil)  │              │   Dynamic WHERE:      │
│   - (nil, err)    │              │   Only filterFields   │
│                   │              │   Dynamic ORDER BY:   │
│                   │              │   Only sortFields     │
└───────────────────┘              └───────────────────────┘
        │
        ↓
┌───────────────────┐              ┌───────────────────────┐
│  4. Search()      │              │   Uses:               │
│  ─────────────    │              │   - selectFields      │
│  Input:           │              │   - searchFields      │
│   ctx context     │              │                       │
│   searchTerm str  │              │   SQL Pattern:        │
│   limit int       │◄─────────────┤   SELECT f1,f2,f3     │
│   offset int      │              │   FROM table          │
│  Output:          │              │   WHERE f1 ILIKE $1   │
│   []*model.Table  │              │      OR f2 ILIKE $2   │
│   error           │              │   LIMIT $x OFFSET $y  │
│                   │              │                       │
│  Returns:         │              │   Special Case:       │
│   - GetAll()      │              │   If no searchFields, │
│     if no search  │              │   fallback to GetAll()│
│   - (items, nil)  │              │                       │
│   - (nil, err)    │              │                       │
└───────────────────┘              └───────────────────────┘
        │
        ↓
┌───────────────────┐              ┌───────────────────────┐
│  5. Update()      │              │   Uses:               │
│  ─────────────    │              │   - updateSets        │
│  Input:           │              │   - updateParams      │
│   ctx context     │              │   - primaryKeyField   │
│   id int64        │              │                       │
│   m *model.Table  │◄─────────────┤   SQL Pattern:        │
│  Output:          │              │   UPDATE table        │
│   error           │              │   SET f1=$1, f2=$2    │
│                   │              │   WHERE pk=$3         │
│  Returns:         │              │                       │
│   - nil if OK     │              │   Check RowsAffected: │
│   - error if      │              │   0 rows = not found  │
│     not found     │              │   error               │
│   - wrapped err   │              │                       │
└───────────────────┘              └───────────────────────┘
        │
        ↓
┌───────────────────┐              ┌───────────────────────┐
│  6. Delete()      │              │   Uses:               │
│  ─────────────    │              │   - primaryKeyField   │
│  Input:           │              │                       │
│   ctx context     │              │   SQL Pattern:        │
│   id int64        │◄─────────────┤   DELETE FROM table   │
│  Output:          │              │   WHERE pk=$1         │
│   error           │              │                       │
│                   │              │   Check RowsAffected: │
│  Returns:         │              │   0 rows = not found  │
│   - nil if OK     │              │   error               │
│   - error if      │              │                       │
│     not found     │              │                       │
│   - wrapped err   │              │                       │
└───────────────────┘              └───────────────────────┘
        │
        ↓
┌───────────────────┐              ┌───────────────────────┐
│  7. Count()       │              │   Uses:               │
│  ─────────────    │              │   - filterFields      │
│  Input:           │              │                       │
│   ctx context     │              │   SQL Pattern:        │
│   filters map     │◄─────────────┤   SELECT COUNT(*)     │
│  Output:          │              │   FROM table          │
│   int64           │              │   [WHERE f1=$1        │
│   error           │              │    AND f2=$2]         │
│                   │              │                       │
│  Returns:         │              │   Dynamic WHERE:      │
│   - (count, nil)  │              │   Only filterFields   │
│   - (0, err)      │              │                       │
└───────────────────┘              └───────────────────────┘
```

---

## Phase 3: Write Repository File

```
┌────────────────────────────────────────────────────────────────┐
│                     File Output Process                        │
└────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────┐
        │   Format Complete Repository Code   │
        │   (fmt.Sprintf with all methods)    │
        └─────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────┐
        │   Filename Pattern:                 │
        │   {schema.Name}_repository.go       │
        │                                     │
        │   Examples:                         │
        │   - user_profile_repository.go      │
        │   - example_table_repository.go     │
        └─────────────────────────────────────┘
                              ↓
        ┌─────────────────────────────────────┐
        │   writeFile(filepath, content)      │
        └─────────────────────────────────────┘
                              ↓
                ┌─────────────────────┐
                │   Success?          │
                └─────────────────────┘
                  ↓              ↓
            ┌─────────┐    ┌─────────┐
            │   YES   │    │   NO    │
            └─────────┘    └─────────┘
                  │              │
                  │              ↓
                  │        return fmt.Errorf
                  │        (wrapped error)
                  │              │
                  └──────────────┘
                              ↓
                    Continue to next schema
```

---

## Complete Data Flow Example: user_profile Table

```
INPUT JSON:
{
  "name": "user_profile",
  "columns": [
    {"name": "user_id", "primary_key": true, "query": {"select": true, "filter": true, "sort": true}, "operations": {"create": false, "get": true}},
    {"name": "username", "query": {"select": true, "filter": true, "search": true}, "operations": {"create": true, "update": true, "get": true}},
    {"name": "created_on", "query": {"select": true, "sort": true}, "operations": {"get": true}, "validation": {"autoSet": "now"}}
  ]
}

                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Field Classification                        │
├─────────────────────────────────────────────────────────────────┤
│  selectFields     = ["user_id", "username", "created_on"]       │
│  createFields     = ["username"]                                │
│  createParams     = ["username"]                                │
│  createValues     = ["$1"]                                      │
│  updateFields     = ["username"]                                │
│  updateParams     = ["username"]                                │
│  updateSets       = ["username = $1"]                           │
│  filterFields     = ["user_id", "username"]                     │
│  sortFields       = ["user_id", "created_on"]                   │
│  searchFields     = ["username"]                                │
│  primaryKeyField  = "user_id"                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Generated SQL Queries                      │
├─────────────────────────────────────────────────────────────────┤
│  Create():                                                      │
│    INSERT INTO user_profile (username)                          │
│    VALUES ($1) RETURNING user_id                                │
│                                                                 │
│  GetByID():                                                     │
│    SELECT user_id, username, created_on                         │
│    FROM user_profile WHERE user_id = $1                         │
│                                                                 │
│  GetAll(filters={"username":"john"}, sort="created_on DESC"):   │
│    SELECT user_id, username, created_on                         │
│    FROM user_profile                                            │
│    WHERE username = $1                                          │
│    ORDER BY created_on DESC                                     │
│    LIMIT $2 OFFSET $3                                           │
│                                                                 │
│  Search(term="john"):                                           │
│    SELECT user_id, username, created_on                         │
│    FROM user_profile                                            │
│    WHERE username ILIKE $1                                      │
│    LIMIT $2 OFFSET $3                                           │
│                                                                 │
│  Update(id=5, model):                                           │
│    UPDATE user_profile SET username = $1                        │
│    WHERE user_id = $2                                           │
│                                                                 │
│  Delete(id=5):                                                  │
│    DELETE FROM user_profile WHERE user_id = $1                  │
│                                                                 │
│  Count(filters={"username":"john"}):                            │
│    SELECT COUNT(*) FROM user_profile WHERE username = $1        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                OUTPUT FILE:
          user_profile_repository.go
```

---

## Error Returns & Edge Cases

```
┌────────────────────────────────────────────────────────────────────┐
│                         Error Scenarios                            │
├────────────────────────────────────────────────────────────────────┤
│  1. Directory creation fails                                       │
│     → return fmt.Errorf("failed to create repository dir: %w")     │
│                                                                    │
│  2. File write fails                                               │
│     → return fmt.Errorf("failed to write file %s: %w", path, err)  │
│                                                                    │
│  3. No primary key found                                           │
│     → primaryKeyField = "" (may cause runtime SQL errors)          │
│     → Consider: Add validation and return error                    │
│                                                                    │
│  4. Empty create/update fields                                     │
│     → Valid if table is read-only                                  │
│     → Create/Update methods will have empty placeholders           │
│                                                                    │
│  5. No searchable fields                                           │
│     → Search() falls back to GetAll()                              │
└────────────────────────────────────────────────────────────────────┘
```

---

## Return Values Summary

```
┌──────────────────────┬────────────────────────────────────────────┐
│ Function             │ Return Values                              │
├──────────────────────┼────────────────────────────────────────────┤
│ GenerateRepoLayer()  │ error (nil if all files written)           │
│                      │                                            │
│ Create()             │ error                                      │
│                      │  - nil: success                            │
│                      │  - wrapped error: DB/query error           │
│                      │                                            │
│ GetByID()            │ (*model.Table, error)                      │
│                      │  - (model, nil): found                     │
│                      │  - (nil, nil): not found                   │
│                      │  - (nil, error): DB error                  │
│                      │                                            │
│ GetAll()             │ ([]*model.Table, error)                    │
│                      │  - (items, nil): success (may be empty)    │
│                      │  - (nil, error): DB error                  │
│                      │                                            │
│ Search()             │ ([]*model.Table, error)                    │
│                      │  - (items, nil): success (may be empty)    │
│                      │  - (nil, error): DB error                  │
│                      │                                            │
│ Update()             │ error                                      │
│                      │  - nil: success                            │
│                      │  - error("not found"): 0 rows affected     │
│                      │  - wrapped error: DB error                 │
│                      │                                            │
│ Delete()             │ error                                      │
│                      │  - nil: success                            │
│                      │  - error("not found"): 0 rows affected     │
│                      │  - wrapped error: DB error                 │
│                      │                                            │
│ Count()              │ (int64, error)                             │
│                      │  - (count, nil): success                   │
│                      │  - (0, error): DB error                    │
└──────────────────────┴────────────────────────────────────────────┘
```

---

## Validation & Security Considerations

```
┌────────────────────────────────────────────────────────────────────┐
│                    Built-in Protections                            │
├────────────────────────────────────────────────────────────────────┤
│  1. Filter Validation                                              │
│     ✓ Only filterFields can be used in WHERE clauses              │
│     ✓ Prevents arbitrary SQL injection via filter keys            │
│                                                                    │
│  2. Sort Validation                                                │
│     ✓ Only sortFields can be used in ORDER BY                     │
│     ✓ Checks for "DESC" and "ASC" suffixes                        │
│                                                                    │
│  3. Parameterized Queries                                          │
│     ✓ All values use $1, $2, $3 placeholders                      │
│     ✓ No string concatenation of user input                       │
│                                                                    │
│  4. Context Support                                                │
│     ✓ All methods accept context.Context                          │
│     ✓ Enables timeout/cancellation                                │
│                                                                    │
│  5. Wrapped Errors                                                 │
│     ✓ All errors wrapped with fmt.Errorf(..., %w)                 │
│     ✓ Maintains error chain for debugging                         │
└────────────────────────────────────────────────────────────────────┘
```


