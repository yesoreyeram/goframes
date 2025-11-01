# Getting Started with GoFrames

## Introduction

Welcome to GoFrames! This guide will help you get started with the GoFrames library, covering installation, basic concepts, and common operations.

## Prerequisites

- Go 1.20 or later
- Basic understanding of Go programming
- Familiarity with data manipulation concepts (helpful but not required)

## Installation

### Using Go Modules (Recommended)

```bash
# Initialize your project (if not already done)
go mod init myproject

# Add GoFrames dependency
go get github.com/yesoreyeram/goframes

# Import in your code
import "github.com/yesoreyeram/goframes"
```

### Verify Installation

```go
package main

import (
    "fmt"
    "github.com/yesoreyeram/goframes"
)

func main() {
    // Create a simple DataFrame
    df, err := goframes.New(map[string]interface{}{
        "name": []string{"Alice", "Bob"},
        "age":  []int{25, 30},
    })
    if err != nil {
        panic(err)
    }
    
    fmt.Println(df)
}
```

## Core Concepts

### DataFrame

A DataFrame is a 2-dimensional labeled data structure with columns of potentially different types.

**Key Properties:**
- Rows and columns
- Column-oriented storage
- Labeled axes (index and columns)
- Immutable by default

### Series

A Series is a 1-dimensional labeled array capable of holding any data type.

**Key Properties:**
- Single data type
- Labeled index
- Vectorized operations

### Index

An Index is used for axis labeling and alignment.

**Types:**
- RangeIndex (default)
- Int64Index
- StringIndex
- DateTimeIndex
- MultiIndex

## Creating DataFrames

### From Map

```go
// Create DataFrame from map of slices
df, err := goframes.New(map[string]interface{}{
    "name":   []string{"Alice", "Bob", "Charlie"},
    "age":    []int{25, 30, 35},
    "city":   []string{"NYC", "LA", "SF"},
    "salary": []float64{75000, 85000, 95000},
})
if err != nil {
    log.Fatal(err)
}

fmt.Println(df)
// Output:
//      name  age  city  salary
// 0   Alice   25   NYC   75000
// 1     Bob   30    LA   85000
// 2 Charlie   35    SF   95000
```

### From Records

```go
// Create DataFrame from slice of maps
records := []map[string]interface{}{
    {"name": "Alice", "age": 25, "city": "NYC"},
    {"name": "Bob", "age": 30, "city": "LA"},
    {"name": "Charlie", "age": 35, "city": "SF"},
}

df, err := goframes.NewFromRecords(records)
if err != nil {
    log.Fatal(err)
}
```

### From CSV File

```go
// Read from CSV file
df, err := goframes.ReadCSV("data.csv")
if err != nil {
    log.Fatal(err)
}

// With options
df, err := goframes.ReadCSV("data.csv",
    goframes.WithHeader(true),
    goframes.WithDelimiter(','),
    goframes.WithSkipRows(1),
)
```

### From JSON

```go
// Read JSON file
df, err := goframes.ReadJSON("data.json")
if err != nil {
    log.Fatal(err)
}

// Read JSON Lines format
df, err := goframes.ReadJSONLines("data.jsonl")
```

### Empty DataFrame

```go
// Create empty DataFrame
df := goframes.Empty()

// Add columns later
df, err = df.AddColumn("name", goframes.NewSeries([]string{"Alice", "Bob"}))
df, err = df.AddColumn("age", goframes.NewSeries([]int{25, 30}))
```

## Basic Operations

### Viewing Data

```go
// Display DataFrame
fmt.Println(df)

// Get shape (rows, columns)
rows, cols := df.Shape()
fmt.Printf("Shape: %d rows, %d columns\n", rows, cols)

// Get first n rows
head := df.Head(5)
fmt.Println(head)

// Get last n rows
tail := df.Tail(5)
fmt.Println(tail)

// Get column names
columns := df.Columns()
fmt.Println("Columns:", columns)

// Get data types
dtypes := df.DTypes()
for col, dtype := range dtypes {
    fmt.Printf("%s: %s\n", col, dtype)
}

// Get summary statistics
summary := df.Describe()
fmt.Println(summary)
```

### Selecting Data

#### Selecting Columns

```go
// Select single column (returns Series)
ages := df.Column("age")
fmt.Println(ages)

// Select multiple columns (returns DataFrame)
subset := df.Select("name", "age")
fmt.Println(subset)
```

#### Selecting Rows

```go
// Select by position (iloc)
row := df.ILoc(0, nil)      // First row, all columns
rows := df.ILoc([]int{0, 2, 4}, nil)  // Multiple rows

// Select by label (loc)
row := df.Loc("row_label", nil)

// Select range
rows := df.ILoc([]int{0, 1, 2}, nil)  // First 3 rows
```

#### Selecting Rows and Columns

```go
// Select specific rows and columns
subset := df.ILoc([]int{0, 1}, []string{"name", "age"})

// Select single cell
value, err := df.At("row_label", "column_name")
value, err := df.IAT(0, 0)  // By position
```

### Filtering Data

```go
// Filter rows based on condition
adults := df.Filter(func(row interface{}) bool {
    return row.(map[string]interface{})["age"].(int) >= 18
})

// Using query string (planned feature)
adults := df.Query("age >= 18")

// Multiple conditions
result := df.Filter(func(row interface{}) bool {
    age := row.(map[string]interface{})["age"].(int)
    city := row.(map[string]interface{})["city"].(string)
    return age >= 25 && city == "NYC"
})
```

### Adding/Removing Columns

```go
// Add new column
df, err = df.AddColumn("age_squared", 
    df.Column("age").Apply(func(x interface{}) interface{} {
        age := x.(int)
        return age * age
    }))

// Remove columns
df, err = df.DropColumn("age_squared")
df, err = df.DropColumn("col1", "col2", "col3")  // Multiple columns

// Rename column
df, err = df.RenameColumn("age", "years")

// Rename multiple columns
df, err = df.RenameColumns(map[string]string{
    "age":  "years",
    "name": "full_name",
})
```

### Sorting

```go
// Sort by single column
sorted := df.Sort([]string{"age"}, []bool{true})  // ascending

// Sort by multiple columns
sorted := df.Sort(
    []string{"city", "age"}, 
    []bool{true, false},  // city ascending, age descending
)

// Sort by index
sorted := df.SortIndex(true)
```

## Working with Series

### Creating Series

```go
// Create from slice
s := goframes.NewSeries([]int{1, 2, 3, 4, 5})

// With custom index
s := goframes.NewSeriesWithIndex(
    []int{10, 20, 30},
    goframes.NewStringIndex([]string{"a", "b", "c"}),
)

// With name
s := goframes.NewSeries(
    []float64{1.1, 2.2, 3.3},
    goframes.WithName("values"),
)
```

### Series Operations

```go
// Arithmetic operations
s1 := goframes.NewSeries([]int{1, 2, 3})
s2 := goframes.NewSeries([]int{4, 5, 6})

sum := s1.Add(s2)       // {5, 7, 9}
diff := s1.Sub(s2)      // {-3, -3, -3}
prod := s1.Mul(s2)      // {4, 10, 18}
quot := s1.Div(s2)      // {0.25, 0.4, 0.5}

// Comparison operations
gt := s1.Gt(2)          // {false, false, true}
eq := s1.Eq(2)          // {false, true, false}

// Aggregations
sum := s.Sum()
mean := s.Mean()
median := s.Median()
min := s.Min()
max := s.Max()
std := s.Std()

// Transformations
doubled := s.Apply(func(x interface{}) interface{} {
    return x.(int) * 2
})

// Null handling
s = s.FillNA(0)         // Fill nulls with 0
s = s.DropNA()          // Remove null values
```

## Aggregations

### Simple Aggregations

```go
// Column-wise aggregations
sums := df.Sum(0, true)      // Sum each column, skip NA
means := df.Mean(0, true)    // Mean of each column
counts := df.Count(0)        // Count non-null values

// Row-wise aggregations
rowSums := df.Sum(1, true)   // Sum each row
```

### GroupBy Aggregations

```go
// Group by single column
grouped := df.GroupBy("city")

// Aggregate
cityStats := grouped.Mean()  // Mean of numeric columns per city
cityCounts := grouped.Count() // Count rows per city
citySums := grouped.Sum()    // Sum per city

// Custom aggregations
cityStats := grouped.Agg(map[string]goframes.AggFunc{
    "age":    goframes.Mean,
    "salary": goframes.Sum,
})

// Group by multiple columns
grouped := df.GroupBy([]string{"city", "department"})
stats := grouped.Mean()
```

## Joining and Merging

### Join DataFrames

```go
// Create two DataFrames
users := goframes.New(map[string]interface{}{
    "user_id": []int{1, 2, 3},
    "name":    []string{"Alice", "Bob", "Charlie"},
})

orders := goframes.New(map[string]interface{}{
    "user_id": []int{1, 2, 2, 3},
    "amount":  []float64{100, 200, 150, 300},
})

// Inner join
result := users.Join(orders, "inner", []string{"user_id"})

// Left join
result := users.Join(orders, "left", []string{"user_id"})

// Merge with different column names
result := users.Merge(orders, "left",
    goframes.JoinOn{Left: "id", Right: "user_id"},
)
```

### Concatenation

```go
// Concatenate DataFrames vertically (stack rows)
combined := df1.Concat([]*goframes.DataFrame{df2, df3}, 0, "outer")

// Concatenate horizontally (add columns)
combined := df1.Concat([]*goframes.DataFrame{df2}, 1, "outer")

// Append rows
df = df.Append(newRow, true)  // ignoreIndex=true
```

## Data Transformation

### Apply Functions

```go
// Apply to column
df, err = df.AddColumn("age_category",
    df.Column("age").Apply(func(x interface{}) interface{} {
        age := x.(int)
        if age < 30 {
            return "young"
        }
        return "senior"
    }))

// Apply to entire DataFrame
transformed := df.Apply(func(row interface{}) interface{} {
    // Transform each row
    return processRow(row)
}, 0)
```

### Replace Values

```go
// Replace specific values
df = df.Replace(nil, 0)  // Replace nulls with 0

// Replace using map
df = df.Column("status").Map(map[interface{}]interface{}{
    "active":   1,
    "inactive": 0,
})
```

### Type Conversion

```go
// Convert column types
df, err = df.AsType(map[string]goframes.DataType{
    "age":    goframes.Int64,
    "salary": goframes.Float64,
})
```

## Handling Missing Data

### Detecting Missing Data

```go
// Check for null values
hasNulls := df.IsNA()  // Returns DataFrame of booleans

// Count nulls per column
nullCounts := df.IsNA().Sum(0, false)
```

### Filling Missing Data

```go
// Fill with constant value
filled := df.FillNA(0)

// Forward fill
filled := df.FillNA(nil, "ffill")

// Backward fill
filled := df.FillNA(nil, "bfill")
```

### Dropping Missing Data

```go
// Drop rows with any null values
cleaned := df.DropNA(0, "any", 0)

// Drop rows where all values are null
cleaned := df.DropNA(0, "all", 0)

// Drop rows with less than N non-null values
cleaned := df.DropNA(0, "any", 3)
```

## Exporting Data

### To CSV

```go
// Write to CSV
err := df.ToCSV("output.csv")

// With options
err := df.ToCSV("output.csv",
    goframes.WithIndex(true),
    goframes.WithHeader(true),
    goframes.WithDelimiter(','),
)
```

### To JSON

```go
// Write to JSON
err := df.ToJSON("output.json")

// JSON Lines format
err := df.ToJSONLines("output.jsonl")
```

### To Other Formats

```go
// Write to Parquet
err := df.ToParquet("output.parquet")

// Write to Excel
err := df.ToExcel("output.xlsx", 
    goframes.WithSheetName("Data"))

// Write to SQL database
err := df.ToSQL("table_name", db,
    goframes.WithIfExists("replace"),
)
```

## Best Practices

### 1. Error Handling

```go
// Always check errors
df, err := goframes.ReadCSV("data.csv")
if err != nil {
    log.Fatalf("Failed to read CSV: %v", err)
}
```

### 2. Use Method Chaining

```go
// Chain operations for clarity
result := df.
    Filter(predicate).
    Select("name", "age", "salary").
    Sort([]string{"salary"}, []bool{false}).
    Head(10)
```

### 3. Pre-allocate When Possible

```go
// If you know the size, pre-allocate
data := make([]int, 0, expectedSize)
for i := 0; i < expectedSize; i++ {
    data = append(data, i)
}
```

### 4. Use Appropriate Data Types

```go
// Choose the smallest type that fits your data
// Use int32 instead of int64 if values fit
// Use float32 instead of float64 if precision allows
df, err := goframes.New(map[string]interface{}{
    "small_int": []int32{1, 2, 3},  // Not []int64
    "price":     []float64{1.99, 2.99},  // Needs precision
})
```

### 5. Profile Performance

```go
import "runtime/pprof"

// CPU profiling
f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()

// Your code here
df.GroupBy("category").Agg(map[string]goframes.AggFunc{
    "value": goframes.Sum,
})

// Analyze with: go tool pprof cpu.prof
```

## Next Steps

- Explore [Advanced Operations](04-advanced-operations.md)
- Learn about [Performance Optimization](03-performance-optimization.md)
- Review [API Reference](02-api-reference.md)
- Check out [System Design](01-system-design.md)

## Example: Complete Workflow

```go
package main

import (
    "fmt"
    "log"
    "github.com/yesoreyeram/goframes"
)

func main() {
    // 1. Load data
    df, err := goframes.ReadCSV("sales.csv")
    if err != nil {
        log.Fatal(err)
    }
    
    // 2. Explore data
    fmt.Println("Shape:", df.Shape())
    fmt.Println("Columns:", df.Columns())
    fmt.Println(df.Head(5))
    
    // 3. Clean data
    df = df.DropNA(0, "any", 0)
    
    // 4. Transform data
    df, err = df.AddColumn("revenue",
        df.Column("price").Mul(df.Column("quantity")))
    
    // 5. Analyze data
    summary := df.GroupBy("category").Agg(map[string]goframes.AggFunc{
        "revenue": goframes.Sum,
        "quantity": goframes.Mean,
    })
    
    // 6. Sort results
    summary = summary.Sort(
        []string{"revenue"}, 
        []bool{false},  // descending
    )
    
    // 7. Export results
    err = summary.ToCSV("summary.csv")
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Println("Analysis complete!")
    fmt.Println(summary.Head(10))
}
```
