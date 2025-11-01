# Testing and Quality Assurance

## Overview

This document outlines the comprehensive testing strategy for GoFrames, including unit tests, integration tests, performance benchmarks, and quality metrics.

## Testing Philosophy

### Core Principles

1. **Comprehensive Coverage**: Aim for >85% line coverage, >80% branch coverage
2. **Test Pyramid**: Many unit tests, fewer integration tests, targeted e2e tests
3. **Performance Regression**: Continuous benchmarking to prevent slowdowns
4. **Property-Based Testing**: Verify invariants hold for arbitrary inputs
5. **Fuzz Testing**: Discover edge cases automatically

## Test Structure

### Directory Layout

```
goframes/
├── core/
│   ├── dataframe.go
│   ├── dataframe_test.go
│   ├── series.go
│   └── series_test.go
├── ops/
│   ├── groupby.go
│   ├── groupby_test.go
│   ├── join.go
│   └── join_test.go
├── io/
│   ├── csv.go
│   ├── csv_test.go
│   └── testdata/
│       └── sample.csv
├── integration_test.go
└── benchmark_test.go
```

## Unit Testing

### Basic Unit Test Structure

```go
package core

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestDataFrameCreation(t *testing.T) {
    // Arrange
    data := map[string]interface{}{
        "name": []string{"Alice", "Bob"},
        "age":  []int{25, 30},
    }
    
    // Act
    df, err := New(data)
    
    // Assert
    require.NoError(t, err)
    assert.Equal(t, 2, df.Len())
    assert.Equal(t, 2, len(df.Columns()))
}
```

### Table-Driven Tests

```go
func TestDataFrameFilter(t *testing.T) {
    tests := []struct {
        name      string
        input     *DataFrame
        predicate Predicate
        expected  int
        wantErr   bool
    }{
        {
            name: "filter greater than",
            input: createTestDF(map[string]interface{}{
                "value": []int{1, 5, 10, 15},
            }),
            predicate: func(row Row) bool {
                return row["value"].(int) > 5
            },
            expected: 2,
            wantErr:  false,
        },
        {
            name: "filter empty result",
            input: createTestDF(map[string]interface{}{
                "value": []int{1, 2, 3},
            }),
            predicate: func(row Row) bool {
                return row["value"].(int) > 10
            },
            expected: 0,
            wantErr:  false,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := tt.input.Filter(tt.predicate)
            
            if tt.wantErr {
                assert.Error(t, err)
                return
            }
            
            require.NoError(t, err)
            assert.Equal(t, tt.expected, result.Len())
        })
    }
}
```

### Test Fixtures

```go
// test_helpers.go
package core

func createTestDataFrame() *DataFrame {
    df, _ := New(map[string]interface{}{
        "id":     []int{1, 2, 3, 4, 5},
        "name":   []string{"Alice", "Bob", "Charlie", "David", "Eve"},
        "age":    []int{25, 30, 35, 40, 45},
        "salary": []float64{75000, 85000, 95000, 105000, 115000},
    })
    return df
}

func createLargeDataFrame(rows int) *DataFrame {
    ids := make([]int, rows)
    values := make([]float64, rows)
    for i := 0; i < rows; i++ {
        ids[i] = i
        values[i] = float64(i)
    }
    df, _ := New(map[string]interface{}{
        "id":    ids,
        "value": values,
    })
    return df
}
```

### Testing Error Cases

```go
func TestDataFrameErrors(t *testing.T) {
    t.Run("empty column name", func(t *testing.T) {
        df := createTestDataFrame()
        _, err := df.AddColumn("", NewSeries([]int{1, 2, 3}))
        assert.Error(t, err)
        assert.Contains(t, err.Error(), "column name cannot be empty")
    })
    
    t.Run("length mismatch", func(t *testing.T) {
        df := createTestDataFrame()
        _, err := df.AddColumn("new", NewSeries([]int{1, 2}))  // Wrong length
        assert.Error(t, err)
        assert.Contains(t, err.Error(), "length mismatch")
    })
    
    t.Run("column not found", func(t *testing.T) {
        df := createTestDataFrame()
        _, err := df.Column("nonexistent")
        assert.Error(t, err)
        assert.Equal(t, ErrColumnNotFound, err)
    })
}
```

## Integration Testing

### Multi-Component Tests

```go
func TestETLPipeline(t *testing.T) {
    // Test complete workflow
    t.Run("CSV read -> transform -> aggregate -> write", func(t *testing.T) {
        // Read
        df, err := ReadCSV("testdata/sales.csv")
        require.NoError(t, err)
        
        // Transform
        df, err = df.AddColumn("revenue", 
            df.Column("price").Mul(df.Column("quantity")))
        require.NoError(t, err)
        
        // Aggregate
        summary := df.GroupBy("category").Agg(map[string]AggFunc{
            "revenue":  Sum,
            "quantity": Mean,
        })
        
        // Write
        tempFile := t.TempDir() + "/output.csv"
        err = summary.ToCSV(tempFile)
        require.NoError(t, err)
        
        // Verify
        result, err := ReadCSV(tempFile)
        require.NoError(t, err)
        assert.True(t, result.Len() > 0)
    })
}
```

### Join Operation Tests

```go
func TestJoinIntegration(t *testing.T) {
    users := createUsersDataFrame()
    orders := createOrdersDataFrame()
    
    tests := []struct {
        name     string
        how      string
        expected int
    }{
        {"inner join", "inner", 5},
        {"left join", "left", 10},
        {"right join", "right", 8},
        {"outer join", "outer", 13},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := users.Join(orders, tt.how, []string{"user_id"})
            require.NoError(t, err)
            assert.Equal(t, tt.expected, result.Len())
        })
    }
}
```

## Property-Based Testing

### Using rapid (Go's property testing framework)

```go
import "testing/quick"

func TestSeriesAdditionCommutative(t *testing.T) {
    f := func(a, b []int) bool {
        if len(a) == 0 || len(b) == 0 || len(a) != len(b) {
            return true
        }
        
        s1 := NewSeries(a)
        s2 := NewSeries(b)
        
        // Test commutativity: a + b == b + a
        result1 := s1.Add(s2)
        result2 := s2.Add(s1)
        
        return seriesEqual(result1, result2)
    }
    
    if err := quick.Check(f, nil); err != nil {
        t.Error(err)
    }
}

func TestSeriesAdditionAssociative(t *testing.T) {
    f := func(a, b, c []int) bool {
        if len(a) == 0 || len(a) != len(b) || len(a) != len(c) {
            return true
        }
        
        s1 := NewSeries(a)
        s2 := NewSeries(b)
        s3 := NewSeries(c)
        
        // Test associativity: (a + b) + c == a + (b + c)
        result1 := s1.Add(s2).Add(s3)
        result2 := s1.Add(s2.Add(s3))
        
        return seriesEqual(result1, result2)
    }
    
    if err := quick.Check(f, nil); err != nil {
        t.Error(err)
    }
}
```

### DataFrame Invariants

```go
func TestDataFrameInvariants(t *testing.T) {
    t.Run("all columns same length", func(t *testing.T) {
        f := func(data map[string][]int) bool {
            if len(data) == 0 {
                return true
            }
            
            // Convert to interface{} slices
            converted := make(map[string]interface{})
            for k, v := range data {
                converted[k] = v
            }
            
            df, err := New(converted)
            if err != nil {
                return true  // Skip invalid inputs
            }
            
            // Verify all columns have same length
            expectedLen := df.Len()
            for _, col := range df.Columns() {
                if df.Column(col).Len() != expectedLen {
                    return false
                }
            }
            return true
        }
        
        if err := quick.Check(f, nil); err != nil {
            t.Error(err)
        }
    })
}
```

## Fuzz Testing

### Fuzz Test Example

```go
func FuzzParseCSV(f *testing.F) {
    // Seed corpus
    f.Add("name,age\nAlice,25\nBob,30\n")
    f.Add("a,b,c\n1,2,3\n4,5,6\n")
    
    f.Fuzz(func(t *testing.T, input string) {
        // Create temp file
        tmpfile, err := os.CreateTemp("", "fuzz-*.csv")
        if err != nil {
            t.Skip()
        }
        defer os.Remove(tmpfile.Name())
        
        if _, err := tmpfile.Write([]byte(input)); err != nil {
            t.Skip()
        }
        tmpfile.Close()
        
        // Try to parse (should not panic)
        _, _ = ReadCSV(tmpfile.Name())
    })
}

func FuzzDataFrameFilter(f *testing.F) {
    // Seed values
    f.Add(int64(0), int64(100))
    f.Add(int64(-100), int64(100))
    
    f.Fuzz(func(t *testing.T, threshold int64) {
        df := createTestDataFrame()
        
        // Should not panic
        _, _ = df.Filter(func(row Row) bool {
            return row["age"].(int) > int(threshold)
        })
    })
}
```

## Benchmark Testing

### Basic Benchmarks

```go
func BenchmarkDataFrameFilter(b *testing.B) {
    df := createLargeDataFrame(1_000_000)
    predicate := func(row Row) bool {
        return row["value"].(float64) > 500_000
    }
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        df.Filter(predicate)
    }
}

func BenchmarkDataFrameGroupBy(b *testing.B) {
    df := createTestDataFrame()
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        df.GroupBy("category").Sum()
    }
}
```

### Parallel Benchmarks

```go
func BenchmarkDataFrameFilterParallel(b *testing.B) {
    df := createLargeDataFrame(1_000_000)
    predicate := func(row Row) bool {
        return row["value"].(float64) > 500_000
    }
    
    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            df.Filter(predicate)
        }
    })
}
```

### Memory Benchmarks

```go
func BenchmarkDataFrameMemory(b *testing.B) {
    b.ReportAllocs()
    
    for i := 0; i < b.N; i++ {
        df := createLargeDataFrame(100_000)
        _ = df.GroupBy("category").Sum()
    }
}
```

### Comparative Benchmarks

```go
func BenchmarkJoinAlgorithms(b *testing.B) {
    df1 := createLargeDataFrame(100_000)
    df2 := createLargeDataFrame(100_000)
    
    b.Run("HashJoin", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            HashJoin(df1, df2, "id")
        }
    })
    
    b.Run("SortMergeJoin", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            SortMergeJoin(df1, df2, "id")
        }
    })
    
    b.Run("NestedLoopJoin", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            NestedLoopJoin(df1, df2, "id")
        }
    })
}
```

## Performance Regression Testing

### Benchmark Tracking

```go
// benchmark_test.go
type BenchmarkResult struct {
    Name     string
    NsPerOp  int64
    AllocsPerOp int64
    BytesPerOp  int64
}

func TestBenchmarkRegression(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping regression test in short mode")
    }
    
    current := runBenchmarks()
    baseline := loadBaseline("benchmark_baseline.json")
    
    for name, curr := range current {
        base, ok := baseline[name]
        if !ok {
            continue
        }
        
        // Check for >5% regression
        if float64(curr.NsPerOp) > float64(base.NsPerOp)*1.05 {
            t.Errorf("Performance regression in %s: %d ns/op vs baseline %d ns/op",
                name, curr.NsPerOp, base.NsPerOp)
        }
        
        // Check for >10% memory increase
        if float64(curr.BytesPerOp) > float64(base.BytesPerOp)*1.10 {
            t.Errorf("Memory regression in %s: %d bytes/op vs baseline %d bytes/op",
                name, curr.BytesPerOp, base.BytesPerOp)
        }
    }
}
```

## Test Coverage

### Coverage Commands

```bash
# Run tests with coverage
go test -coverprofile=coverage.out ./...

# View coverage report
go tool cover -html=coverage.out

# Coverage by package
go test -coverprofile=coverage.out ./... && \
  go tool cover -func=coverage.out

# Minimum coverage threshold
go test -coverprofile=coverage.out ./... && \
  go tool cover -func=coverage.out | \
  grep total | \
  awk '{if ($3 < 85.0) exit 1}'
```

### Coverage Configuration

```go
// .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-go@v2
        with:
          go-version: '1.20'
      - name: Run tests
        run: go test -v -race -coverprofile=coverage.out ./...
      - name: Check coverage
        run: |
          coverage=$(go tool cover -func=coverage.out | \
            grep total | awk '{print $3}' | sed 's/%//')
          if (( $(echo "$coverage < 85" | bc -l) )); then
            echo "Coverage $coverage% is below 85%"
            exit 1
          fi
```

## Quality Metrics

### Code Quality Checks

```bash
# Static analysis
go vet ./...

# Linting
golangci-lint run

# Cyclomatic complexity
gocyclo -over 15 .

# Code duplication
dupl -threshold 100 .

# Security scanning
gosec ./...
```

### CI/CD Integration

```yaml
# .github/workflows/quality.yml
name: Quality
on: [push, pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-go@v2
      - name: Vet
        run: go vet ./...
      - name: Lint
        uses: golangci/golangci-lint-action@v2
        with:
          version: latest
      - name: Security
        uses: securego/gosec@master
        with:
          args: './...'
```

## Test Organization Best Practices

### 1. Naming Conventions

```go
// Test functions
func TestFunctionName(t *testing.T)           // Unit test
func TestIntegrationWorkflow(t *testing.T)    // Integration test
func BenchmarkFunctionName(b *testing.B)      // Benchmark
func FuzzFunctionName(f *testing.F)           // Fuzz test
func ExampleFunctionName()                     // Example (documentation)

// Test cases
func TestDataFrame_AddColumn_Success(t *testing.T)
func TestDataFrame_AddColumn_EmptyName(t *testing.T)
func TestDataFrame_AddColumn_LengthMismatch(t *testing.T)
```

### 2. Test Data Management

```go
// testdata/ directory structure
testdata/
├── csv/
│   ├── valid.csv
│   ├── invalid_delimiter.csv
│   └── missing_headers.csv
├── json/
│   ├── valid.json
│   └── malformed.json
└── fixtures/
    └── large_dataset.parquet
```

### 3. Parallel Test Execution

```go
func TestParallelOperations(t *testing.T) {
    t.Parallel()  // Enable parallel execution
    
    t.Run("filter", func(t *testing.T) {
        t.Parallel()
        // Test filter
    })
    
    t.Run("sort", func(t *testing.T) {
        t.Parallel()
        // Test sort
    })
}
```

### 4. Cleanup and Resource Management

```go
func TestWithTempFile(t *testing.T) {
    tmpDir := t.TempDir()  // Automatically cleaned up
    
    file := filepath.Join(tmpDir, "test.csv")
    df.ToCSV(file)
    
    // Test with file
    result, err := ReadCSV(file)
    require.NoError(t, err)
    
    // No manual cleanup needed
}
```

## Continuous Testing

### Pre-commit Hooks

```bash
# .git/hooks/pre-commit
#!/bin/bash

# Run tests
go test ./... || exit 1

# Run linters
golangci-lint run || exit 1

# Check formatting
unformatted=$(gofmt -l .)
if [ -n "$unformatted" ]; then
    echo "Unformatted files:"
    echo "$unformatted"
    exit 1
fi
```

### Nightly Test Suite

```yaml
# .github/workflows/nightly.yml
name: Nightly Tests
on:
  schedule:
    - cron: '0 0 * * *'  # Midnight daily
jobs:
  extended-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Extended test suite
        run: go test -v -timeout 1h ./...
      - name: Performance benchmarks
        run: go test -bench=. -benchtime=10s ./...
      - name: Fuzz testing
        run: go test -fuzz=. -fuzztime=1h ./...
```

## Documentation Testing

### Example Tests

```go
// Example tests serve as documentation
func ExampleDataFrame_Filter() {
    df, _ := New(map[string]interface{}{
        "age": []int{25, 30, 35},
    })
    
    result, _ := df.Filter(func(row Row) bool {
        return row["age"].(int) > 25
    })
    
    fmt.Println(result.Len())
    // Output: 2
}
```

## Summary

A comprehensive testing strategy ensures GoFrames is:
- **Reliable**: Bugs caught early through extensive testing
- **Performant**: Benchmarks prevent regressions
- **Maintainable**: Clear tests serve as documentation
- **Robust**: Fuzz testing discovers edge cases
- **Production-Ready**: High coverage and quality metrics
