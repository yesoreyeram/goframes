# Data Types and Type System

## Overview

GoFrames provides a rich type system designed to efficiently represent various data types commonly found in data analysis while leveraging Go's static typing for safety and performance.

## Type Hierarchy

```
DataType (interface)
├── Primitive Types
│   ├── Boolean
│   ├── Numeric
│   │   ├── Integer
│   │   │   ├── Int8, Int16, Int32, Int64
│   │   │   └── UInt8, UInt16, UInt32, UInt64
│   │   └── FloatingPoint
│   │       ├── Float32
│   │       └── Float64
│   └── String
├── Temporal Types
│   ├── DateTime
│   ├── Date
│   ├── Time
│   └── Duration
├── Complex Types
│   ├── Struct
│   ├── List
│   └── Map
├── Special Types
│   ├── Null
│   └── Category
└── Binary Types
    └── Binary
```

## Primitive Data Types

### Boolean Type

**Memory Layout:**
- 1 bit per value (packed in bitmap)
- 8 values per byte
- Extremely memory efficient

**Operations:**
- Logical: AND, OR, NOT, XOR
- Comparison: EQ, NE
- Aggregation: Any, All, Sum (count of True)

```go
// Create boolean series
bools := goframes.NewSeries([]bool{true, false, true, true})

// Operations
result := bools.And(other)  // Element-wise AND
result := bools.Or(other)   // Element-wise OR
result := bools.Not()       // Element-wise NOT

// Aggregations
anyTrue := bools.Any()      // true if any value is true
allTrue := bools.All()      // true if all values are true
count := bools.Sum()        // count of true values
```

### Integer Types

**Signed Integers:**
- **Int8**: -128 to 127 (1 byte)
- **Int16**: -32,768 to 32,767 (2 bytes)
- **Int32**: -2,147,483,648 to 2,147,483,647 (4 bytes)
- **Int64**: -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 (8 bytes)

**Unsigned Integers:**
- **UInt8**: 0 to 255 (1 byte)
- **UInt16**: 0 to 65,535 (2 bytes)
- **UInt32**: 0 to 4,294,967,295 (4 bytes)
- **UInt64**: 0 to 18,446,744,073,709,551,615 (8 bytes)

**Best Practices:**
```go
// Choose smallest type that fits your data
ages := goframes.NewSeries([]int8{25, 30, 35})        // Ages fit in Int8
years := goframes.NewSeries([]int16{2020, 2021, 2022}) // Years fit in Int16
ids := goframes.NewSeries([]int64{...})                // Large IDs need Int64

// Memory savings example:
// 1M ages as Int64: 8 MB
// 1M ages as Int8:  1 MB (8x savings)
```

**Operations:**
- Arithmetic: +, -, *, /, %
- Comparison: <, <=, >, >=, ==, !=
- Bitwise: &, |, ^, <<, >>
- Aggregation: Sum, Mean, Min, Max, Std, Var

### Floating Point Types

**Types:**
- **Float32**: IEEE 754 single precision (4 bytes, ~7 decimal digits)
- **Float64**: IEEE 754 double precision (8 bytes, ~15 decimal digits)

**Special Values:**
```go
import "math"

// Special float64 values
posInf := math.Inf(1)    // Positive infinity
negInf := math.Inf(-1)   // Negative infinity
nan := math.NaN()        // Not a Number

// Check special values
math.IsInf(value, 0)     // Is infinity
math.IsNaN(value)        // Is NaN
```

**Precision Considerations:**
```go
// Float32 precision limits
var f32 float32 = 1.0 + 1e-8  // May lose precision
var f64 float64 = 1.0 + 1e-15 // Maintains precision

// Use Float64 for:
// - Financial calculations
// - Scientific computing
// - Accumulating values

// Use Float32 for:
// - Graphics/rendering
// - ML features (where precision less critical)
// - Memory-constrained environments
```

### String Type

**Implementation:**
- UTF-8 encoded
- Variable length
- Immutable
- Pointer to string data + length

**Memory Optimization:**
```go
// Dictionary encoding for low-cardinality strings
// Original: ["red", "blue", "red", "green", "blue", "red"]
// Dictionary: {0: "red", 1: "blue", 2: "green"}
// Encoded: [0, 1, 0, 2, 1, 0]
// Memory: ~50% reduction for this example

// Use Category type for low-cardinality strings
categories := goframes.NewCategoricalSeries(
    []string{"red", "blue", "red", "green"},
)
```

**String Operations:**
```go
s := df.Column("text")

// String methods
s.StrContains("pattern")     // Contains substring
s.StrStartsWith("prefix")    // Starts with prefix
s.StrEndsWith("suffix")      // Ends with suffix
s.StrUpper()                 // Convert to uppercase
s.StrLower()                 // Convert to lowercase
s.StrLen()                   // String length
s.StrReplace("old", "new")   // Replace substring
s.StrSplit(",")              // Split by delimiter
```

## Temporal Data Types

### DateTime Type

**Representation:**
- Stored as int64 nanoseconds since epoch
- Timezone-aware optional
- Resolution: nanosecond

**Creation:**
```go
import "time"

// Create DateTime series
dates := goframes.NewSeries([]time.Time{
    time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC),
    time.Date(2024, 1, 2, 0, 0, 0, 0, time.UTC),
})

// Parse from strings
dates, err := goframes.ParseDates(
    []string{"2024-01-01", "2024-01-02"},
    layout: "2006-01-02",
)
```

**DateTime Operations:**
```go
dt := df.Column("timestamp")

// Extract components
years := dt.DateYear()       // Extract year
months := dt.DateMonth()     // Extract month
days := dt.DateDay()         // Extract day
hours := dt.DateHour()       // Extract hour
weekdays := dt.DateWeekday() // Day of week (0-6)

// Formatting
formatted := dt.StrFormat("2006-01-02")

// Time arithmetic
shifted := dt.Add(time.Hour * 24)  // Add 1 day
diff := dt.Sub(other)              // Time difference
```

### Duration Type

**Representation:**
- Stored as int64 nanoseconds
- Represents time span

```go
// Create duration series
durations := goframes.NewSeries([]time.Duration{
    time.Hour,
    time.Hour * 2,
    time.Minute * 30,
})

// Convert to different units
hours := durations.AsHours()
minutes := durations.AsMinutes()
seconds := durations.AsSeconds()
```

## Null Handling

### Null Representation

**Bitmap Approach:**
```
Data:     [10, 20, 30, 40, 50]
Nulls:    [ 0,  1,  0,  0,  1]  (1 = null, 0 = valid)
Result:   [10, NA, 30, 40, NA]
```

**Memory Efficiency:**
- Separate null bitmap
- 1 bit per value
- Only allocated if nulls exist

### Null Operations

```go
s := df.Column("values")

// Check for nulls
isNull := s.IsNA()      // Boolean series
notNull := s.NotNA()    // Boolean series
hasNulls := s.HasNulls() // Single boolean
count := s.NullCount()   // Count of nulls

// Fill nulls
filled := s.FillNA(0)              // Fill with value
filled := s.FillNA(nil, "ffill")   // Forward fill
filled := s.FillNA(nil, "bfill")   // Backward fill

// Drop nulls
clean := s.DropNA()

// Null-aware operations
sum := s.Sum(skipNA: true)   // Skip nulls in aggregation
```

## Complex Data Types

### Struct Type

**Use Case:** Nested/hierarchical data

```go
// Define struct type
type Person struct {
    Name    string
    Age     int
    Address Address
}

type Address struct {
    Street string
    City   string
}

// Create struct series
people := goframes.NewSeries([]Person{
    {Name: "Alice", Age: 25, Address: Address{...}},
    {Name: "Bob", Age: 30, Address: Address{...}},
})

// Access nested fields
names := people.StructField("Name")
cities := people.StructField("Address.City")
```

### List Type

**Use Case:** Array-valued columns

```go
// Create list series
lists := goframes.NewSeries([][]int{
    {1, 2, 3},
    {4, 5},
    {6, 7, 8, 9},
})

// List operations
lengths := lists.ListLen()          // Length of each list
flattened := lists.ListFlatten()    // Flatten to single series
element := lists.ListGet(0)         // Get first element of each list
```

### Map Type

**Use Case:** Key-value pairs

```go
// Create map series
maps := goframes.NewSeries([]map[string]int{
    {"a": 1, "b": 2},
    {"c": 3, "d": 4},
})

// Map operations
keys := maps.MapKeys()          // Get all keys
values := maps.MapValues()      // Get all values
lookup := maps.MapGet("a")      // Get value for key
```

## Special Types

### Category Type

**Purpose:** Efficient storage of low-cardinality string data

**Implementation:**
```
Original: ["red", "blue", "red", "green", "blue", "red"]  (6 strings)
Categories: ["red", "blue", "green"]                       (3 strings)
Codes:      [0, 1, 0, 2, 1, 0]                            (6 integers)
```

**Memory Savings:**
- Original: 6 strings × average 4 bytes = ~24 bytes + overhead
- Categorical: 3 strings + 6 int8s = ~12 bytes + overhead
- Savings: ~50% for this example, more for larger datasets

**Usage:**
```go
// Create categorical series
cat := goframes.NewCategoricalSeries(
    []string{"red", "blue", "red", "green", "blue"},
)

// Category operations
categories := cat.Categories()           // Get unique categories
ordered := cat.AsOrdered()               // Mark as ordered
cat = cat.AddCategory("yellow")          // Add new category
cat = cat.RemoveUnusedCategories()       // Clean up
cat = cat.ReorderCategories(order)       // Change order

// Benefits
// 1. Memory efficient
// 2. Faster groupby operations
// 3. Preserves order if needed
// 4. Type safety for limited values
```

## Type Conversion

### Automatic Type Promotion

**Rules:**
```
Int8 + Int16   → Int16
Int32 + Int64  → Int64
Int + Float    → Float
Float32 + Float64 → Float64
```

**Example:**
```go
s1 := goframes.NewSeries([]int32{1, 2, 3})
s2 := goframes.NewSeries([]int64{4, 5, 6})

// Automatic promotion to int64
result := s1.Add(s2)  // Result is int64 series
```

### Explicit Conversion

```go
// Convert series type
s := goframes.NewSeries([]int{1, 2, 3})

// To float64
floats, err := s.AsType(goframes.Float64)

// To string
strings, err := s.AsType(goframes.String)

// DataFrame type conversion
df, err = df.AsType(map[string]goframes.DataType{
    "age":    goframes.Int32,
    "salary": goframes.Float64,
    "name":   goframes.String,
})
```

### Parsing and Casting

```go
// Parse strings to numbers
nums, err := goframes.ParseInt([]string{"1", "2", "3"})
floats, err := goframes.ParseFloat([]string{"1.1", "2.2"})

// Parse dates
dates, err := goframes.ParseDates(
    []string{"2024-01-01", "2024-01-02"},
    layout: "2006-01-02",
)

// With error handling for invalid values
nums, err := goframes.ParseIntWithErrors(
    []string{"1", "invalid", "3"},
    errorValue: math.NaN(),
)
```

## Type Inference

### Automatic Type Detection

```go
// When reading from CSV, types are inferred
df, err := goframes.ReadCSV("data.csv")
// Infers:
// - Integer columns as Int64
// - Decimal columns as Float64
// - Other columns as String
// - Date-like strings as DateTime

// Inference rules:
// 1. Try parsing as integer
// 2. Try parsing as float
// 3. Try parsing as datetime
// 4. Default to string
```

### Custom Type Hints

```go
// Override type inference
df, err := goframes.ReadCSV("data.csv",
    goframes.WithDTypes(map[string]goframes.DataType{
        "id":     goframes.Int64,
        "price":  goframes.Float64,
        "date":   goframes.DateTime,
        "status": goframes.Category,
    }),
)
```

## Memory Layout

### Physical Storage

**Integer Example:**
```
Series: [1, 2, 3, 4, 5]
Memory: [0x01, 0x02, 0x03, 0x04, 0x05] (contiguous, 1 byte each for Int8)
```

**String Example:**
```
Series: ["hello", "world"]
Memory:
  Offsets: [0, 5, 10]        (start positions)
  Data:    "helloworld"       (concatenated)
  
Access "hello": data[offsets[0]:offsets[1]]
Access "world": data[offsets[1]:offsets[2]]
```

**Null Example:**
```
Series: [1, NA, 3, 4, NA]
Data:   [1, 0, 3, 4, 0]      (0 for null, actual value doesn't matter)
Nulls:  [0, 1, 0, 0, 1]      (bitmap: 1 = null)
```

### Alignment and Padding

```go
// Memory alignment for performance
// Aligned: Faster access, slightly more memory
// Packed:  Less memory, potentially slower

// Default: 8-byte alignment for best performance
type AlignedInt64 struct {
    data [1000000]int64  // Naturally aligned
}

// Memory usage:
// - 1M int64 values = 8 MB
// - Null bitmap = 125 KB
// - Total ≈ 8.125 MB
```

## Best Practices

### 1. Choose Appropriate Types

```go
// ✓ Good: Use smallest type that fits
ages := []int8{25, 30, 35}

// ✗ Bad: Waste memory
ages := []int64{25, 30, 35}  // 8x more memory than needed
```

### 2. Use Categories for Low-Cardinality Data

```go
// ✓ Good: Category for repeated values
status := goframes.NewCategoricalSeries(statuses)

// ✗ Bad: String for repeated values
status := goframes.NewSeries(statuses)  // Much more memory
```

### 3. Be Explicit About Nulls

```go
// ✓ Good: Handle nulls explicitly
result := df.Column("value").Sum(skipNA: true)

// ✗ Bad: Ignore null handling
result := df.Column("value").Sum()  // May include nulls
```

### 4. Consider Timezone for DateTime

```go
// ✓ Good: Explicit timezone
dates := df.Column("timestamp").TzLocalize("UTC")

// ✗ Bad: Naive timestamps
dates := df.Column("timestamp")  // Ambiguous timezone
```

### 5. Validate Type Conversions

```go
// ✓ Good: Check conversion errors
floats, err := ints.AsType(goframes.Float64)
if err != nil {
    log.Fatal(err)
}

// ✗ Bad: Ignore errors
floats, _ := ints.AsType(goframes.Float64)
```

## Type Registry

### Custom Type Registration

```go
// Register custom type
type MyCustomType struct {
    // ...
}

func (mct MyCustomType) Name() string { return "MyCustomType" }
func (mct MyCustomType) Size() int { return 16 }
// ... implement DataType interface

// Register
goframes.RegisterDataType(MyCustomType{})

// Use
df, err := goframes.New(map[string]interface{}{
    "custom": []MyCustomType{...},
})
```
