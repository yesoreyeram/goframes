# Advanced Data Operations

## Overview

This document covers advanced data manipulation operations in GoFrames, including joins, groupby, pivots, window functions, and time series operations.

## Join Operations

### Types of Joins

GoFrames supports all standard SQL join types:

1. **Inner Join**: Returns only matching rows
2. **Left Join**: Returns all left rows + matching right rows
3. **Right Join**: Returns all right rows + matching left rows
4. **Outer Join**: Returns all rows from both DataFrames
5. **Cross Join**: Cartesian product of both DataFrames

### Join Syntax

```go
// Basic join
result := df1.Join(df2, "inner", []string{"id"})

// Join with different column names
result := df1.Merge(df2, "left", 
    JoinOn{Left: "user_id", Right: "id"},
    WithSuffixes("_user", "_profile"))

// Multiple key join
result := df1.Join(df2, "outer", []string{"year", "month", "day"})
```

### Join Algorithms

#### 1. Hash Join (Default for Equi-joins)

**Algorithm:**
```
1. Build hash table from smaller DataFrame
2. Probe larger DataFrame against hash table
3. Emit matching rows
```

**Characteristics:**
- **Time Complexity**: O(n + m)
- **Space Complexity**: O(min(n, m))
- **Best For**: Large datasets with good hash distribution
- **Worst Case**: High hash collisions

**Implementation Strategy:**
```go
type HashJoin struct {
    buildTable map[interface{}][]int
    buildDF    *DataFrame
    probeDF    *DataFrame
}

func (hj *HashJoin) Execute() *DataFrame {
    // Phase 1: Build hash table
    for i := 0; i < hj.buildDF.Len(); i++ {
        key := hj.buildDF.GetKeyValue(i)
        hj.buildTable[key] = append(hj.buildTable[key], i)
    }
    
    // Phase 2: Probe and emit
    results := []Row{}
    for i := 0; i < hj.probeDF.Len(); i++ {
        key := hj.probeDF.GetKeyValue(i)
        if matches, ok := hj.buildTable[key]; ok {
            for _, matchIdx := range matches {
                results = append(results, 
                    combineRows(hj.probeDF.GetRow(i), 
                               hj.buildDF.GetRow(matchIdx)))
            }
        }
    }
    
    return NewDataFrame(results)
}
```

#### 2. Sort-Merge Join

**Algorithm:**
```
1. Sort both DataFrames on join keys
2. Merge sorted streams
3. Emit matching rows
```

**Characteristics:**
- **Time Complexity**: O(n log n + m log m)
- **Space Complexity**: O(1) for pre-sorted data
- **Best For**: Pre-sorted data or range joins
- **Advantage**: Streaming-friendly

#### 3. Nested Loop Join

**Characteristics:**
- **Time Complexity**: O(n * m)
- **Space Complexity**: O(1)
- **Best For**: Small datasets or complex join conditions
- **Use Case**: Non-equi joins, cross joins

### Join Optimization

**Optimization Strategies:**

1. **Smaller DataFrame as Build Side**
   ```go
   if df1.Len() < df2.Len() {
       buildDF, probeDF = df1, df2
   } else {
       buildDF, probeDF = df2, df1
   }
   ```

2. **Index-based Joins**
   ```go
   // If join column is indexed
   if df2.Index().Column() == joinKey {
       return IndexJoin(df1, df2, joinKey)
   }
   ```

3. **Parallel Hash Partitioning**
   ```go
   // Partition both DataFrames by hash
   partitions1 := hashPartition(df1, numPartitions)
   partitions2 := hashPartition(df2, numPartitions)
   
   // Join each partition pair in parallel
   results := make(chan *DataFrame, numPartitions)
   for i := 0; i < numPartitions; i++ {
       go func(idx int) {
           results <- join(partitions1[idx], partitions2[idx])
       }(i)
   }
   ```

## GroupBy Operations

### GroupBy Mechanics

**Process:**
1. **Grouping**: Partition rows by key(s)
2. **Aggregation**: Apply function to each group
3. **Combining**: Merge results into DataFrame

### Single-Key GroupBy

```go
// Group by single column
grouped := df.GroupBy("category")

// Aggregations
result := grouped.Sum()          // Sum all numeric columns
result := grouped.Mean()         // Mean of numeric columns
result := grouped.Count()        // Count rows per group
result := grouped.Agg(map[string]AggFunc{
    "sales": Sum,
    "price": Mean,
    "quantity": Max,
})
```

### Multi-Key GroupBy

```go
// Group by multiple columns
grouped := df.GroupBy([]string{"year", "month", "category"})

// Result has hierarchical index
result := grouped.Sum()
// Index: MultiIndex[(2024, 1, "Electronics"), (2024, 1, "Books"), ...]
```

### Custom Aggregation Functions

```go
// Define custom aggregation
func percentile90(s *Series) interface{} {
    sorted := s.Sort(true)
    idx := int(float64(s.Len()) * 0.9)
    return sorted.IAT(idx)
}

// Use custom aggregation
result := grouped.Agg(map[string]AggFunc{
    "response_time": percentile90,
    "error_rate": Mean,
})
```

### GroupBy Algorithms

#### 1. Hash-based GroupBy (Default)

```go
type HashGroupBy struct {
    groups map[interface{}][]int  // key -> row indices
}

func (hgb *HashGroupBy) Build(df *DataFrame, keys []string) {
    for i := 0; i < df.Len(); i++ {
        key := extractKey(df, i, keys)
        hgb.groups[key] = append(hgb.groups[key], i)
    }
}

func (hgb *HashGroupBy) Aggregate(df *DataFrame, aggFunc AggFunc) *DataFrame {
    results := make([]Row, 0, len(hgb.groups))
    for key, indices := range hgb.groups {
        groupDF := df.ILoc(indices, nil)
        aggResult := aggFunc(groupDF)
        results = append(results, Row{Key: key, Value: aggResult})
    }
    return NewDataFrame(results)
}
```

**Complexity:** O(n) for grouping, O(n) for aggregation

#### 2. Sort-based GroupBy

```go
// 1. Sort by group keys
sortedDF := df.Sort(keys, ascending)

// 2. Sequential scan to identify groups
currentKey := nil
groupStart := 0
for i := 0; i <= sortedDF.Len(); i++ {
    if i == sortedDF.Len() || extractKey(sortedDF, i) != currentKey {
        if currentKey != nil {
            // Process group [groupStart:i]
            processGroup(sortedDF.ILoc(groupStart:i, nil))
        }
        currentKey = extractKey(sortedDF, i)
        groupStart = i
    }
}
```

**Complexity:** O(n log n) for sorting, O(n) for aggregation
**Advantage:** Memory efficient, streaming-friendly

### Transform and Filter

```go
// Transform: Apply function returning same-shaped data
result := grouped.Transform(func(df *DataFrame) *DataFrame {
    mean := df.Column("value").Mean()
    return df.AddColumn("normalized", 
        df.Column("value").Sub(mean))
})

// Filter: Keep only groups matching condition
result := grouped.Filter(func(df *DataFrame) bool {
    return df.Len() > 10  // Keep groups with >10 rows
})
```

## Pivot and Reshape Operations

### Pivot Tables

**Concept:** Reshape data from long to wide format

```go
// Long format
// date, product, sales
// 2024-01-01, A, 100
// 2024-01-01, B, 150
// 2024-01-02, A, 120

// Pivot
result := df.Pivot(
    index: "date",
    columns: "product",
    values: "sales",
    aggFunc: Sum,
)

// Wide format
// date, A, B
// 2024-01-01, 100, 150
// 2024-01-02, 120, NaN
```

### Pivot Implementation

```go
func (df *DataFrame) Pivot(index, columns, values string, aggFunc AggFunc) *DataFrame {
    // 1. Group by index + columns
    grouped := df.GroupBy([]string{index, columns})
    
    // 2. Aggregate values
    aggregated := grouped.Agg(map[string]AggFunc{values: aggFunc})
    
    // 3. Reshape to wide format
    uniqueColumns := df.Column(columns).Unique()
    uniqueIndices := df.Column(index).Unique()
    
    result := NewDataFrame()
    for _, idx := range uniqueIndices {
        row := map[string]interface{}{index: idx}
        for _, col := range uniqueColumns {
            value := aggregated.Loc(
                Row: idx,
                Col: col,
            )
            row[col] = value
        }
        result.AddRow(row)
    }
    
    return result
}
```

### Melt (Unpivot)

**Concept:** Reshape data from wide to long format

```go
// Wide format
// date, A, B, C
// 2024-01-01, 100, 150, 200
// 2024-01-02, 120, 160, 210

// Melt
result := df.Melt(
    idVars: []string{"date"},
    valueVars: []string{"A", "B", "C"},
    varName: "product",
    valueName: "sales",
)

// Long format
// date, product, sales
// 2024-01-01, A, 100
// 2024-01-01, B, 150
// 2024-01-01, C, 200
// 2024-01-02, A, 120
```

### Stack and Unstack

```go
// Stack: Pivot level(s) of columns to rows
stacked := df.Stack(level: -1)  // Stack innermost level

// Unstack: Pivot level(s) of index to columns
unstacked := df.Unstack(level: 0)  // Unstack outermost level
```

## Window Functions

### Rolling Windows

**Fixed-size sliding window operations**

```go
// Rolling window
rolling := df.Rolling(window: 7)  // 7-day window

// Aggregations
result := rolling.Mean()    // 7-day moving average
result := rolling.Sum()     // 7-day rolling sum
result := rolling.Std()     // 7-day rolling std dev
result := rolling.Min()     // 7-day rolling min
result := rolling.Max()     // 7-day rolling max

// Custom window function
result := rolling.Apply(func(window *DataFrame) interface{} {
    // Custom calculation on window
    return window.Column("value").Quantile(0.75)
})
```

### Rolling Implementation

```go
func (df *DataFrame) Rolling(window int) *Rolling {
    return &Rolling{
        df:     df,
        window: window,
        minPeriods: window,
    }
}

func (r *Rolling) Mean() *DataFrame {
    result := make([]float64, r.df.Len())
    
    for i := 0; i < r.df.Len(); i++ {
        start := max(0, i-r.window+1)
        end := i + 1
        
        if end-start < r.minPeriods {
            result[i] = math.NaN()
        } else {
            windowData := r.df.ILoc(start:end, nil)
            result[i] = windowData.Column(r.column).Mean()
        }
    }
    
    return r.df.AddColumn("rolling_mean", NewSeries(result))
}
```

### Expanding Windows

**Cumulative operations from start**

```go
// Expanding window (all data up to current row)
expanding := df.Expanding(minPeriods: 1)

result := expanding.Sum()     // Cumulative sum
result := expanding.Mean()    // Cumulative average
result := expanding.Count()   // Cumulative count
```

### Exponentially Weighted Windows

**Time-decay weighted operations**

```go
// Exponentially weighted moving average
ewm := df.EWM(span: 10)  // Equivalent to 10-period SMA
result := ewm.Mean()

// With half-life
ewm := df.EWM(halflife: 7.0)  // 7-day half-life
result := ewm.Mean()
```

## Time Series Operations

### Date Range Operations

```go
// Create date range
dates := NewDateRange(
    start: "2024-01-01",
    end: "2024-12-31",
    freq: "D",  // Daily
)

// Business days only
dates := NewDateRange(
    start: "2024-01-01",
    end: "2024-12-31",
    freq: "B",  // Business days
)
```

### Resampling

**Change frequency of time series data**

```go
// Downsample (reduce frequency)
daily := df.Resample("D").Mean()      // Daily average
weekly := df.Resample("W").Sum()      // Weekly sum
monthly := df.Resample("M").Last()    // Month-end values

// Upsample (increase frequency)
hourly := df.Resample("H").Interpolate()  // Hourly with interpolation
```

### Shifting and Lagging

```go
// Shift forward (lag)
df.AddColumn("prev_value", df.Column("value").Shift(1))

// Shift backward (lead)
df.AddColumn("next_value", df.Column("value").Shift(-1))

// Shift with frequency
df.Shift(1, freq: "D")  // Shift by 1 day
```

### Time-based Indexing

```go
// Set datetime index
df = df.SetIndex(NewDateTimeIndex(df.Column("date")))

// Select by date range
result := df.Loc("2024-01-01":"2024-12-31")
result := df.Loc("2024-Q1")  // First quarter
result := df.Loc("2024-01")  // January 2024

// Truncate
result := df.Truncate(before: "2024-01-01", after: "2024-12-31")
```

### Time Zone Handling

```go
// Convert timezone
df.Column("timestamp").TzConvert("America/New_York")

// Localize (add timezone to naive timestamps)
df.Column("timestamp").TzLocalize("UTC")

// Remove timezone
df.Column("timestamp").TzLocalize(nil)
```

## Advanced Filtering

### Complex Predicates

```go
// Multiple conditions
result := df.Filter(func(row Row) bool {
    return row["age"] > 18 && 
           row["country"] == "USA" &&
           row["active"] == true
})

// Using query strings
result := df.Query("age > 18 and country == 'USA' and active")
```

### Conditional Selection

```go
// Where - replace values not matching condition
result := df.Where(
    condition: df.Column("value") > 0,
    other: 0,  // Replace non-matching with 0
)

// Mask - replace values matching condition
result := df.Mask(
    condition: df.Column("value") < 0,
    other: 0,  // Replace matching with 0
)
```

## Data Alignment

### Automatic Alignment

```go
// DataFrames with different indices align automatically
df1 := NewDataFrame(
    data: map[string][]int{"value": {1, 2, 3}},
    index: NewStringIndex([]string{"a", "b", "c"}),
)

df2 := NewDataFrame(
    data: map[string][]int{"value": {4, 5, 6}},
    index: NewStringIndex([]string{"b", "c", "d"}),
)

// Addition aligns on index
result := df1.Add(df2)
// Index: ["a", "b", "c", "d"]
// Values: [NaN, 6, 8, NaN]
```

### Reindexing

```go
// Reindex with new index
result := df.Reindex(NewStringIndex([]string{"x", "y", "z"}))

// Fill missing values
result := df.Reindex(
    NewStringIndex([]string{"x", "y", "z"}),
    fillValue: 0,
)

// Forward fill
result := df.Reindex(
    NewStringIndex([]string{"x", "y", "z"}),
    method: "ffill",
)
```

## Categorical Data

### Category Type

**Efficient storage for repetitive string data**

```go
// Create categorical column
df.AddColumn("category", 
    NewCategoricalSeries([]string{"A", "B", "A", "C", "B", "A"}))

// Memory: 3 unique strings + 6 integer codes
// vs. 6 full strings

// Operations preserve categories
filtered := df.Filter(df.Column("category").Eq("A"))
```

### Category Operations

```go
// Get categories
categories := df.Column("category").Categories()

// Add category
df.Column("category").AddCategory("D")

// Remove unused categories
df.Column("category").RemoveUnusedCategories()

// Reorder categories
df.Column("category").ReorderCategories([]string{"C", "B", "A"})

// Set categories as ordered
df.Column("category").AsOrdered()
```

## Multi-Index Operations

### Creating Multi-Index

```go
// From arrays
index := NewMultiIndex(
    levels: []Index{
        NewStringIndex([]string{"2024", "2025"}),
        NewStringIndex([]string{"Q1", "Q2", "Q3", "Q4"}),
    },
    codes: [][]int{
        {0, 0, 0, 0, 1, 1, 1, 1},  // Year codes
        {0, 1, 2, 3, 0, 1, 2, 3},  // Quarter codes
    },
)

// From DataFrame columns
df.SetIndex([]string{"year", "quarter"})
```

### Multi-Index Selection

```go
// Select by tuple
result := df.Loc((2024, "Q1"))

// Select by slice
result := df.Loc((2024, slice("Q1", "Q3")))

// Cross-section
result := df.XS("Q1", level: "quarter")

// Partial indexing
result := df.Loc(2024)  // All quarters in 2024
```

### Multi-Index Aggregation

```go
// Group by level
result := df.GroupBy(level: 0).Sum()  // Sum by year
result := df.GroupBy(level: "quarter").Mean()  // Mean by quarter

// Swap levels
result := df.SwapLevel(0, 1)

// Sort by index
result := df.SortIndex(level: [0, 1])
```
