# GoFrames API Reference

## Overview

This document provides a comprehensive reference for the GoFrames API, including all proposed interfaces, types, and methods for the DataFrame library.

## Package Structure

```
goframes/
├── core/           # Core DataFrame and Series implementations
├── io/             # Input/Output operations
├── ops/            # Operations (groupby, join, pivot, etc.)
├── types/          # Type system and data types
├── index/          # Index implementations
├── compute/        # Computation engine
├── storage/        # Storage and memory management
└── utils/          # Utility functions
```

## Core Types

### DataFrame

The primary data structure for 2D labeled data.

```go
package core

type DataFrame struct {
    // Private fields
}

// Creation Functions
func New(data map[string]interface{}) (*DataFrame, error)
func NewFromRecords(records []map[string]interface{}) (*DataFrame, error)
func NewFromColumns(columns map[string]*Series) (*DataFrame, error)
func Empty() *DataFrame
func Range(start, stop, step int) *DataFrame

// Properties
func (df *DataFrame) Shape() (rows, cols int)
func (df *DataFrame) Columns() []string
func (df *DataFrame) Index() *Index
func (df *DataFrame) DTypes() map[string]DataType
func (df *DataFrame) Size() int
func (df *DataFrame) Empty() bool
func (df *DataFrame) Memory() int64

// Selection and Indexing
func (df *DataFrame) Loc(rows, cols interface{}) (*DataFrame, error)
func (df *DataFrame) ILoc(rows, cols interface{}) (*DataFrame, error)
func (df *DataFrame) At(row, col interface{}) (interface{}, error)
func (df *DataFrame) IAT(row, col int) (interface{}, error)
func (df *DataFrame) Select(columns ...string) (*DataFrame, error)
func (df *DataFrame) Filter(predicate Predicate) (*DataFrame, error)
func (df *DataFrame) Head(n int) *DataFrame
func (df *DataFrame) Tail(n int) *DataFrame
func (df *DataFrame) Sample(n int, options ...SampleOption) *DataFrame
func (df *DataFrame) Query(expr string) (*DataFrame, error)

// Column Operations
func (df *DataFrame) AddColumn(name string, series *Series) (*DataFrame, error)
func (df *DataFrame) DropColumn(names ...string) (*DataFrame, error)
func (df *DataFrame) RenameColumn(old, new string) (*DataFrame, error)
func (df *DataFrame) RenameColumns(mapping map[string]string) (*DataFrame, error)
func (df *DataFrame) SelectDTypes(include, exclude []DataType) (*DataFrame, error)

// Row Operations
func (df *DataFrame) AddRow(data map[string]interface{}) (*DataFrame, error)
func (df *DataFrame) DropRow(indices ...int) (*DataFrame, error)
func (df *DataFrame) DropDuplicates(subset []string, keep string) (*DataFrame, error)
func (df *DataFrame) DropNA(axis int, how string, thresh int) (*DataFrame, error)
func (df *DataFrame) FillNA(value interface{}, method string) (*DataFrame, error)

// Transformations
func (df *DataFrame) Apply(fn ApplyFunc, axis int) (*DataFrame, error)
func (df *DataFrame) ApplyMap(fn MapFunc) (*DataFrame, error)
func (df *DataFrame) Map(mapping map[interface{}]interface{}) (*DataFrame, error)
func (df *DataFrame) Replace(old, new interface{}) (*DataFrame, error)
func (df *DataFrame) AsType(dtypes map[string]DataType) (*DataFrame, error)

// Sorting
func (df *DataFrame) Sort(by []string, ascending []bool) (*DataFrame, error)
func (df *DataFrame) SortIndex(ascending bool) (*DataFrame, error)
func (df *DataFrame) SortValues(by string, ascending bool) (*DataFrame, error)

// Aggregations
func (df *DataFrame) Sum(axis int, skipNA bool) interface{}
func (df *DataFrame) Mean(axis int, skipNA bool) interface{}
func (df *DataFrame) Median(axis int, skipNA bool) interface{}
func (df *DataFrame) Min(axis int, skipNA bool) interface{}
func (df *DataFrame) Max(axis int, skipNA bool) interface{}
func (df *DataFrame) Std(axis int, skipNA bool) interface{}
func (df *DataFrame) Var(axis int, skipNA bool) interface{}
func (df *DataFrame) Count(axis int) interface{}
func (df *DataFrame) Quantile(q float64, axis int) interface{}
func (df *DataFrame) Describe() *DataFrame

// GroupBy Operations
func (df *DataFrame) GroupBy(by interface{}) *GroupBy
func (df *DataFrame) Pivot(index, columns, values string, aggFunc AggFunc) (*DataFrame, error)
func (df *DataFrame) PivotTable(index, columns, values []string, aggFunc AggFunc) (*DataFrame, error)
func (df *DataFrame) Melt(idVars, valueVars []string) (*DataFrame, error)

// Joining and Merging
func (df *DataFrame) Join(other *DataFrame, how string, on []string) (*DataFrame, error)
func (df *DataFrame) Merge(other *DataFrame, how string, on []string, suffixes [2]string) (*DataFrame, error)
func (df *DataFrame) Concat(others []*DataFrame, axis int, join string) (*DataFrame, error)
func (df *DataFrame) Append(other *DataFrame, ignoreIndex bool) (*DataFrame, error)

// Window Operations
func (df *DataFrame) Rolling(window int) *Rolling
func (df *DataFrame) Expanding(minPeriods int) *Expanding
func (df *DataFrame) EWM(span, halflife float64) *ExponentiallyWeighted

// Time Series Operations
func (df *DataFrame) Resample(rule string) *Resampler
func (df *DataFrame) Shift(periods int, freq string) (*DataFrame, error)
func (df *DataFrame) Diff(periods int) (*DataFrame, error)
func (df *DataFrame) PctChange(periods int) (*DataFrame, error)

// Information
func (df *DataFrame) Info() string
func (df *DataFrame) String() string
func (df *DataFrame) Head() string
func (df *DataFrame) Value() interface{}

// Iteration
func (df *DataFrame) IterRows() RowIterator
func (df *DataFrame) IterCols() ColIterator
func (df *DataFrame) IterTuples() TupleIterator

// Copying
func (df *DataFrame) Copy(deep bool) *DataFrame
func (df *DataFrame) Clone() *DataFrame

// Validation
func (df *DataFrame) IsNA() *DataFrame
func (df *DataFrame) NotNA() *DataFrame
func (df *DataFrame) Any() bool
func (df *DataFrame) All() bool
```

### Series

One-dimensional labeled array.

```go
package core

type Series struct {
    // Private fields
}

// Creation Functions
func NewSeries(data interface{}, options ...SeriesOption) (*Series, error)
func NewSeriesWithIndex(data interface{}, index *Index) (*Series, error)
func SeriesFromSlice(data interface{}) (*Series, error)

// Properties
func (s *Series) Name() string
func (s *Series) SetName(name string) *Series
func (s *Series) Len() int
func (s *Series) DType() DataType
func (s *Series) Index() *Index
func (s *Series) Values() interface{}
func (s *Series) HasNulls() bool
func (s *Series) NullCount() int

// Selection and Indexing
func (s *Series) Loc(labels interface{}) (*Series, error)
func (s *Series) ILoc(positions interface{}) (*Series, error)
func (s *Series) At(label interface{}) (interface{}, error)
func (s *Series) IAT(position int) (interface{}, error)
func (s *Series) Get(i int) (interface{}, error)
func (s *Series) Set(i int, value interface{}) error
func (s *Series) Filter(predicate Predicate) (*Series, error)
func (s *Series) Head(n int) *Series
func (s *Series) Tail(n int) *Series
func (s *Series) Sample(n int) *Series

// Transformations
func (s *Series) Apply(fn func(interface{}) interface{}) *Series
func (s *Series) Map(mapping map[interface{}]interface{}) *Series
func (s *Series) Replace(old, new interface{}) *Series
func (s *Series) AsType(dtype DataType) (*Series, error)
func (s *Series) FillNA(value interface{}) *Series
func (s *Series) DropNA() *Series

// Sorting
func (s *Series) Sort(ascending bool) *Series
func (s *Series) SortIndex(ascending bool) *Series
func (s *Series) Argsort(ascending bool) []int

// Aggregations
func (s *Series) Sum() interface{}
func (s *Series) Mean() float64
func (s *Series) Median() float64
func (s *Series) Min() interface{}
func (s *Series) Max() interface{}
func (s *Series) Std() float64
func (s *Series) Var() float64
func (s *Series) Count() int
func (s *Series) Quantile(q float64) interface{}
func (s *Series) Mode() *Series
func (s *Series) Unique() *Series
func (s *Series) NUnique() int
func (s *Series) ValueCounts() *Series

// Comparison Operations
func (s *Series) Eq(other interface{}) *Series
func (s *Series) Ne(other interface{}) *Series
func (s *Series) Lt(other interface{}) *Series
func (s *Series) Le(other interface{}) *Series
func (s *Series) Gt(other interface{}) *Series
func (s *Series) Ge(other interface{}) *Series

// Arithmetic Operations
func (s *Series) Add(other interface{}) *Series
func (s *Series) Sub(other interface{}) *Series
func (s *Series) Mul(other interface{}) *Series
func (s *Series) Div(other interface{}) *Series
func (s *Series) Mod(other interface{}) *Series
func (s *Series) Pow(other interface{}) *Series

// Logical Operations
func (s *Series) And(other *Series) *Series
func (s *Series) Or(other *Series) *Series
func (s *Series) Not() *Series

// String Operations (for String dtype)
func (s *Series) StrContains(pattern string) *Series
func (s *Series) StrStartsWith(prefix string) *Series
func (s *Series) StrEndsWith(suffix string) *Series
func (s *Series) StrUpper() *Series
func (s *Series) StrLower() *Series
func (s *Series) StrLen() *Series
func (s *Series) StrReplace(old, new string) *Series
func (s *Series) StrSplit(sep string) *Series

// DateTime Operations (for DateTime dtype)
func (s *Series) DateYear() *Series
func (s *Series) DateMonth() *Series
func (s *Series) DateDay() *Series
func (s *Series) DateHour() *Series
func (s *Series) DateMinute() *Series
func (s *Series) DateSecond() *Series
func (s *Series) DateWeekday() *Series

// Copying
func (s *Series) Copy() *Series

// Information
func (s *Series) String() string
func (s *Series) Describe() map[string]interface{}
```

### GroupBy

GroupBy operation result.

```go
package ops

type GroupBy struct {
    // Private fields
}

// Aggregation Operations
func (gb *GroupBy) Sum() *DataFrame
func (gb *GroupBy) Mean() *DataFrame
func (gb *GroupBy) Median() *DataFrame
func (gb *GroupBy) Min() *DataFrame
func (gb *GroupBy) Max() *DataFrame
func (gb *GroupBy) Std() *DataFrame
func (gb *GroupBy) Var() *DataFrame
func (gb *GroupBy) Count() *DataFrame
func (gb *GroupBy) Size() *Series
func (gb *GroupBy) First() *DataFrame
func (gb *GroupBy) Last() *DataFrame
func (gb *GroupBy) Nth(n int) *DataFrame

// Custom Aggregations
func (gb *GroupBy) Agg(funcs map[string]AggFunc) *DataFrame
func (gb *GroupBy) Apply(fn ApplyFunc) *DataFrame
func (gb *GroupBy) Transform(fn TransformFunc) *DataFrame
func (gb *GroupBy) Filter(fn FilterFunc) *DataFrame

// Iteration
func (gb *GroupBy) Iter() GroupIterator
func (gb *GroupBy) Groups() map[interface{}][]int

// Properties
func (gb *GroupBy) NGroups() int
func (gb *GroupBy) Indices() map[interface{}][]int
```

### Rolling Window

Rolling window operations.

```go
package ops

type Rolling struct {
    // Private fields
}

func (r *Rolling) Sum() *DataFrame
func (r *Rolling) Mean() *DataFrame
func (r *Rolling) Median() *DataFrame
func (r *Rolling) Min() *DataFrame
func (r *Rolling) Max() *DataFrame
func (r *Rolling) Std() *DataFrame
func (r *Rolling) Var() *DataFrame
func (r *Rolling) Count() *DataFrame
func (r *Rolling) Apply(fn ApplyFunc) *DataFrame
func (r *Rolling) Quantile(q float64) *DataFrame
func (r *Rolling) Corr() *DataFrame
func (r *Rolling) Cov() *DataFrame
```

### Index

Index for axis labeling.

```go
package index

type Index interface {
    Len() int
    Get(i int) interface{}
    GetLoc(label interface{}) (int, error)
    GetIndexer(labels []interface{}) ([]int, error)
    IsUnique() bool
    IsMonotonic() bool
    Equals(other Index) bool
    Union(other Index) Index
    Intersection(other Index) Index
    Difference(other Index) Index
    Slice(start, stop int) Index
    Copy() Index
}

// Concrete Implementations
type RangeIndex struct { /* ... */ }
type Int64Index struct { /* ... */ }
type StringIndex struct { /* ... */ }
type DateTimeIndex struct { /* ... */ }
type MultiIndex struct { /* ... */ }

// Creation Functions
func NewRangeIndex(start, stop, step int) *RangeIndex
func NewInt64Index(data []int64) *Int64Index
func NewStringIndex(data []string) *StringIndex
func NewDateTimeIndex(data []time.Time) *DateTimeIndex
func NewMultiIndex(levels []Index, codes [][]int) *MultiIndex
```

## IO Operations

### Reading Data

```go
package io

// CSV
func ReadCSV(path string, options ...CSVOption) (*DataFrame, error)
func ReadCSVWithSchema(path string, schema *Schema, options ...CSVOption) (*DataFrame, error)

type CSVOption func(*CSVReader)
func WithDelimiter(delim rune) CSVOption
func WithHeader(has bool) CSVOption
func WithSkipRows(n int) CSVOption
func WithNRows(n int) CSVOption
func WithColumns(cols []string) CSVOption
func WithDTypes(dtypes map[string]DataType) CSVOption
func WithNullValues(values []string) CSVOption
func WithCompression(comp string) CSVOption

// JSON
func ReadJSON(path string, options ...JSONOption) (*DataFrame, error)
func ReadJSONLines(path string, options ...JSONOption) (*DataFrame, error)

type JSONOption func(*JSONReader)
func WithOrient(orient string) JSONOption
func WithLines(lines bool) JSONOption

// Parquet
func ReadParquet(path string, options ...ParquetOption) (*DataFrame, error)

type ParquetOption func(*ParquetReader)
func WithColumns(cols []string) ParquetOption
func WithRowGroups(groups []int) ParquetOption

// Excel
func ReadExcel(path string, options ...ExcelOption) (*DataFrame, error)

type ExcelOption func(*ExcelReader)
func WithSheet(name string) ExcelOption
func WithHeader(row int) ExcelOption

// SQL
func ReadSQL(query string, conn *sql.DB) (*DataFrame, error)
func ReadSQLTable(table string, conn *sql.DB) (*DataFrame, error)

// Cloud Storage
func ReadFromS3(bucket, key string, format string) (*DataFrame, error)
func ReadFromGCS(bucket, object string, format string) (*DataFrame, error)
func ReadFromAzure(container, blob string, format string) (*DataFrame, error)
```

### Writing Data

```go
package io

// CSV
func (df *DataFrame) ToCSV(path string, options ...CSVWriteOption) error
func (df *DataFrame) ToCSVStream(writer io.Writer, options ...CSVWriteOption) error

type CSVWriteOption func(*CSVWriter)
func WithIndex(include bool) CSVWriteOption
func WithHeader(include bool) CSVWriteOption
func WithDelimiter(delim rune) CSVWriteOption

// JSON
func (df *DataFrame) ToJSON(path string, options ...JSONWriteOption) error
func (df *DataFrame) ToJSONLines(path string) error

type JSONWriteOption func(*JSONWriter)
func WithOrient(orient string) JSONWriteOption
func WithIndent(indent string) JSONWriteOption

// Parquet
func (df *DataFrame) ToParquet(path string, options ...ParquetWriteOption) error

type ParquetWriteOption func(*ParquetWriter)
func WithCompression(codec string) ParquetWriteOption
func WithRowGroupSize(size int) ParquetWriteOption

// Excel
func (df *DataFrame) ToExcel(path string, options ...ExcelWriteOption) error

type ExcelWriteOption func(*ExcelWriter)
func WithSheetName(name string) ExcelWriteOption

// SQL
func (df *DataFrame) ToSQL(table string, conn *sql.DB, options ...SQLWriteOption) error

type SQLWriteOption func(*SQLWriter)
func WithIfExists(action string) SQLWriteOption
func WithChunkSize(size int) SQLWriteOption

// Cloud Storage
func (df *DataFrame) ToS3(bucket, key string, format string) error
func (df *DataFrame) ToGCS(bucket, object string, format string) error
func (df *DataFrame) ToAzure(container, blob string, format string) error
```

## Type System

```go
package types

type DataType interface {
    Name() string
    Kind() Kind
    Size() int
    IsNumeric() bool
    IsComparable() bool
}

type Kind int

const (
    KindBool Kind = iota
    KindInt8
    KindInt16
    KindInt32
    KindInt64
    KindUInt8
    KindUInt16
    KindUInt32
    KindUInt64
    KindFloat32
    KindFloat64
    KindString
    KindDateTime
    KindDuration
    KindStruct
    KindList
    KindMap
    KindNull
)

// Standard Types
var (
    Bool     DataType
    Int8     DataType
    Int16    DataType
    Int32    DataType
    Int64    DataType
    UInt8    DataType
    UInt16   DataType
    UInt32   DataType
    UInt64   DataType
    Float32  DataType
    Float64  DataType
    String   DataType
    DateTime DataType
    Duration DataType
)

// Type Functions
func InferType(value interface{}) DataType
func ParseType(name string) (DataType, error)
func PromoteTypes(types ...DataType) DataType
```

## Function Types

```go
package core

// Function Types for Operations
type ApplyFunc func(interface{}) interface{}
type MapFunc func(interface{}) interface{}
type AggFunc func(*Series) interface{}
type TransformFunc func(*DataFrame) *DataFrame
type FilterFunc func(*DataFrame) bool
type Predicate func(interface{}) bool

// Comparison Function
type CompareFunc func(a, b interface{}) int

// Iterators
type RowIterator interface {
    Next() bool
    Row() map[string]interface{}
}

type ColIterator interface {
    Next() bool
    Column() (string, *Series)
}

type TupleIterator interface {
    Next() bool
    Tuple() []interface{}
}

type GroupIterator interface {
    Next() bool
    Group() (interface{}, *DataFrame)
}
```

## Options and Configuration

```go
package config

type Config struct {
    MaxParallelism      int
    DefaultChunkSize    int
    MemoryLimit         int64
    EnableLazyEval      bool
    EnableQueryOptimize bool
    DefaultNullValues   []string
}

func DefaultConfig() *Config
func (c *Config) SetMaxParallelism(n int)
func (c *Config) SetMemoryLimit(bytes int64)
func (c *Config) SetLazyEval(enable bool)

// Global configuration
func SetGlobalConfig(cfg *Config)
func GetGlobalConfig() *Config
```

## Error Types

```go
package errors

type DataFrameError struct {
    Op  string
    Err error
}

func (e *DataFrameError) Error() string

var (
    ErrInvalidShape     error
    ErrTypeMismatch     error
    ErrIndexOutOfBounds error
    ErrColumnNotFound   error
    ErrInvalidOperation error
    ErrNullValue        error
    ErrInvalidInput     error
)
```

## Usage Examples

### Creating DataFrames

```go
// From map
df, err := goframes.New(map[string]interface{}{
    "name": []string{"Alice", "Bob", "Charlie"},
    "age":  []int{25, 30, 35},
    "city": []string{"NYC", "LA", "SF"},
})

// From CSV
df, err := goframes.ReadCSV("data.csv")

// From records
records := []map[string]interface{}{
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 30},
}
df, err := goframes.NewFromRecords(records)
```

### Data Selection

```go
// Select columns
subset := df.Select("name", "age")

// Filter rows
adults := df.Filter(func(row interface{}) bool {
    return row.(map[string]interface{})["age"].(int) >= 18
})

// Loc and ILoc
rows := df.Loc([]int{0, 2}, []string{"name", "age"})
rows := df.ILoc([]int{0, 2}, []int{0, 1})
```

### Transformations

```go
// Add column
df = df.AddColumn("age_squared", df.Select("age").Apply(
    func(x interface{}) interface{} {
        age := x.(int)
        return age * age
    }, 0))

// Sort
df = df.Sort([]string{"age"}, []bool{false})

// GroupBy
grouped := df.GroupBy("city").Mean()
```

### Joins and Merges

```go
// Join
result := df1.Join(df2, "inner", []string{"id"})

// Merge with suffixes
result := df1.Merge(df2, "left", []string{"key"}, [2]string{"_x", "_y"})

// Concat
result := df1.Concat([]*DataFrame{df2, df3}, 0, "outer")
```
