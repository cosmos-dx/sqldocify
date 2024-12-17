# Contribution-Guidelines

The folder structure of SQLDocify is - 
```
 ---configs/
 ----------configs.go
 ----------tablemetafunc.go
 ---servers/
 ----------mysql/
 ----------sqlite/
 ----------MoreServersCanBeThere
 ----------index.go
 ---table/
 ----------queries/
 -----------------mysqlqueries.go
 -----------------MoreQueriesCanBeThere
 -----------------factory.go
 -----------------template.go
 -----------------utility.go
 ----------applied.go
 ----------table.go
 ---validators/
```
**Configs** - Here configs.go is a type of global structures or types which are going to be used globally.

**Servers** - Servers are the multiple servers which will be initialised.

**Table** - Here on the basis of server selected we can choose the types of SQL queries

**Validators** - Validators helps to validate data or something.

## Methodology
1. Here we created a method to dynamically select the queries of database which is being used as server.

2. For Effective DB queries and for scalability we introduced a method of **metadata**. Where we store the details of the tables and on that basis we performs the operations.


## Architecture

These are the Common Interface which will have different Query for various Databases.

```
type QueryGenerator interface { // Get schema of a table
	GenerateCreateTableQuery(nm string, schema map[string]configs.FieldSchema) string                                                                                // Single create table
	GenerateGetSchemaQuery(db *sql.DB, nm string) (map[string]configs.FieldSchema, error)                                                                            // Get schema of a table
	GenerateGetAllTablesQuery(db *sql.DB) ([]string, error)                                                                                                          // Get all tables
	GenerateTableExistsQuery(table string) string                                                                                                                    // Single table exists
	GenerateInsertQuery(table string, columns []string, values []interface{}) string                                                                                 // Single insert
	GenerateMultipleInsertQuery(table string, columns []string, values [][]interface{}) string                                                                       // Bulk insert
	GenerateUpdateQuery(table string, updates map[string]interface{}, condition string) string                                                                       // Single update
	GenerateDeleteQuery(table string, condition string) string                                                                                                       // Single delete
	GenerateMultipleDeleteQuery(table string, conditions []string) string                                                                                            // Bulk delete
	GenerateSelectQuery(table string, columns []string, condition string, orderBy string, limit int, offset int) string                                              // Single select
	GenerateBatchInsertQuery(table string, columns []string, batchValues [][]interface{}) string                                                                     // Batch insert
	GenerateUpsertQuery(table string, columns []string, values []interface{}, conflictColumns []string, updates map[string]interface{}) string                       // Single upsert
	GenerateJoinQuery(mainTable string, joinType string, joinTable string, onCondition string, columns []string, condition string, orderBy string, limit int) string // Single join
	GenerateCountQuery(table string, condition string) string                                                                                                        // Single count
	GenerateExistsQuery(table string, condition string) string                                                                                                       // Single exists
	GenerateTransactionQuery(queries []string) string                                                                                                                // Single transaction
	GenerateAggregationQuery(table string, columns []string, aggregations map[string]string, condition string, groupBy []string, orderBy string) string              // Single aggregation
	BuildConditionQuery(conditions map[string]interface{}, logicalOperator string) string                                                                            // Build condition query
	GeneratePaginationQuery(table string, columns []string, condition string, orderBy string, page int, pageSize int) string                                         // Single pagination
	GenerateCreateIndexQuery(indexName string, table string, columns []string, unique bool) string                                                                   // Single create index
	GenerateDropIndexQuery(indexName string) string                                                                                                                  // Single drop index
	GenerateAddColumnQuery(table string, columnName string, columnType string, defaultValue interface{}) string                                                      // Single add column
	GenerateModifyColumnQuery(table string, columnName string, columnType string, nullable bool) string                                                              // Single modify column
	GenerateDropColumnQuery(table string, columnName string) string                                                                                                  // Single drop column
	GenerateAddForeignKeyQuery(table string, columnName string, referencedTable string, referencedColumn string, onDelete string, onUpdate string) string            // Single add foreign key
	GenerateDropForeignKeyQuery(table string, foreignKeyName string) string                                                                                          // Single drop foreign key
	SanitizeValue(value interface{}) string                                                                                                                          // Sanitize value
	FormatColumns(columns []string) string                                                                                                                           // Format columns
	EscapeIdentifier(identifier string) string                                                                                                                       // Escape identifier
}
```

Below is a common interface. table.go

```
type ITableSpec interface {
	GetMetaDataSchema() (interface{}, error)
	TableExists(nm string, db *configs.Database) bool
	CreateTable(db *configs.Database, nm string, schema map[string]configs.FieldSchema) error
	Insert(dt interface{}, db *configs.Database) error
	Update(dt interface{}, onUpdate func() error, db *configs.Database) (UpdateDiffs, error)
	Delete(condition interface{}, db *configs.Database) error
	Fetch(condition interface{}, result interface{}, db *configs.Database) error
	BeginTransaction(db *configs.Database) error
	CommitTransaction(db *configs.Database) error
	RollbackTransaction(db *configs.Database) error
	BatchInsert(dts []interface{}, db *configs.Database) error
	BatchUpdate(dts []interface{}, onUpdate func() error, db *configs.Database) ([]UpdateDiffs, error)
	BatchDelete(conditions []interface{}, db *configs.Database) error
}

```

## metadata

Initialisation of metadata

```
type MetaTableDetails struct {
	Schema    map[string]FieldSchema `json:"schema"`
	Timestamp string                 `json:"timestamp"`
	Details   string                 `json:"details"`
}

type MetaTableList struct {
	mu             sync.Mutex
	ExistingTables map[string]MetaTableDetails `json:"existing_tables"`
}

var (
	instance *MetaTableList
	once     sync.Once
)
```
Implemented Functions 

```
unc GetMetaTableInstance() *MetaTableList {
	once.Do(func() {
		instance = &MetaTableList{
			ExistingTables: make(map[string]MetaTableDetails),
		}
		if err := loadActiveMetaTables(instance); err != nil {
			panic(err)
		}
	})
	return instance
}
func (t *MetaTableList) FindMetaTable(name string) *MetaTableDetails {
	t.mu.Lock()
	defer t.mu.Unlock()

	if details, exists := t.ExistingTables[name]; exists {
		return &details
	}
	return nil
}

func (tl *MetaTableList) UpdateMetaTable(tableName string, details MetaTableDetails) {
	tl.mu.Lock()
	defer tl.mu.Unlock()
	tl.ExistingTables[tableName] = details
	saveActiveMetaTables(tl, "activetables.json")
}

func (tl *MetaTableList) RemoveMetaTable(tableName string) {
	tl.mu.Lock()
	defer tl.mu.Unlock()
	delete(tl.ExistingTables, tableName)
	saveActiveMetaTables(tl, "activetables.json")
}

func loadActiveMetaTables(tl *MetaTableList) error {
	fileName := "activetables.json"

	if _, err := os.Stat(fileName); os.IsNotExist(err) {
		file, err := os.Create(fileName)
		if err != nil {
			return err
		}
		defer file.Close()

		tl.ExistingTables = make(map[string]MetaTableDetails)
		return saveActiveMetaTables(tl, fileName)
	}

	file, err := os.Open(fileName)
	if err != nil {
		return err
	}
	defer file.Close()

	byteValue, err := ioutil.ReadAll(file)
	if err != nil {
		return err
	}

	return json.Unmarshal(byteValue, &tl.ExistingTables)
}

func saveActiveMetaTables(tl *MetaTableList, fileName string) error {
	data, err := json.MarshalIndent(tl.ExistingTables, "", "  ")
	if err != nil {
		return err
	}
	return ioutil.WriteFile(fileName, data, 0644)
}

```

**The usecase of metatable is that we stores the tables details into a file and we check everytime the table if any query is hitted it reduces the overhead on a database.**

## **Issues**

Here issue rely in main.go file or where user will use the library.

    
    func main() {
	dbType := "mysql"
	config := "root:pass@tcp(127.0.0.1:3306)/gotest1"

	db, err := servers.NewDatabase(dbType, config)
	if err != nil {
		log.Fatalf("Failed to connect to the database: %v", err)
	}
	defer func() {
		if err := db.Close(); err != nil {
			log.Printf("Failed to close the database connection: %v", err)
		}
	}()
	// .CreateTables(db)
	// table.TableExists("users", db)
	tablespec, _ := table.AddSelectedDB()
	tablespec.CreateTable(db.DB(), "abc", userTableSchema)
	fmt.Println("Database connection established successfully!")

	fmt.Println("Operations completed.")
    }

    1. User have to use functions from different package everytime while user do not know  about the functions.
    2. While using function for CreateTable he has to initialise itself with a tablespec alone due to the reciever funtion.

## **Future Scope**

In future we want to skip the CreateTable and want to include it in the Schema system. All the functions should be binded with schema. 

Please Contribute it will help a lot.
