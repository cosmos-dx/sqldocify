# SQLDOCIFY

SQLDocify is a versatile GoLang-based SQL library that simplifies database interactions by abstracting complex queries. It enables developers to focus on business logic without managing or debugging raw SQL statements.

With ready-to-use functions and support for multiple databases, SQLDocify allows seamless integration across different backends, ensuring efficient and hassle-free development.

## Initial Setup
Provide dbType  ```mysql, sqlite, postgres ```.

Provide config ```username:password@tcp(127.0.0.1:3306)/your-db-name``` 
```
func main() {
	dbType := "mysql"
	config := "username:password@tcp(127.0.0.1:3306)/golangdb"

	db, err := servers.NewDatabase(dbType, config)
	if err != nil {
		log.Fatalf("Failed to connect to the database: %v", err)
	}
	defer func() {
		if err := db.Close(); err != nil {
			log.Printf("Failed to close the database connection: %v", err)
		}
	}()
	fmt.Println("Database connection established successfully!")

}
```

## Create Table

For table creation you have to provide schema early. Schema Pattern has to be followed strictly.

```
type FieldSchema struct {
	Type  string `json:"Type"`
	Null  string `json:"Null"`
	Key   string `json:"Key"`
	Extra string `json:"Extra"`
}

```
**Here TYPE, NULL, KEY, EXTRA are essential parameters**
```
//Example schema
var userTableSchema = map[string]configs.FieldSchema{
	"email": {
		Type:  "varchar(255)",
		Null:  "NO",
		Key:   "UNI",
		Extra: "",
	},
	"id": {
		Type:  "int",
		Null:  "NO",
		Key:   "PRI",
		Extra: "auto_increment",
	},
	"password": {
		Type:  "varchar(255)",
		Null:  "NO",
		Key:   "",
		Extra: "",
	},
	"status": {
		Type:  "enum('active','inactive','banned')",
		Null:  "NO",
		Key:   "",
		Extra: "",
	},
	"username": {
		Type:  "varchar(255)",
		Null:  "NO",
		Key:   "UNI",
		Extra: "",
	},
}

```

## Functions 
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
# Contribution
We have a scope to correct our errors and make this library more useful and scalable.
This needs your help and we will be really thankful if you contribute and help us to make this library more robust. 

Contribution guidelines link - https://github.com/cosmos-dx/sqldocify/blob/main/contribution-guidelines.md
