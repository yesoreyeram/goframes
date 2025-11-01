# GoFrames

**Enterprise-grade DataFrame library for Go**

GoFrames is a high-performance, type-safe DataFrame library for Go, designed to provide data manipulation capabilities similar to Python's pandas, but optimized for Go's strengths in concurrency, memory efficiency, and static typing.

## Overview

GoFrames combines the familiar DataFrame paradigm with Go's performance and type safety, making it ideal for production data processing pipelines, ETL workflows, and data-intensive applications.

### Key Features

- **Type Safety**: Leverage Go's static typing to catch errors at compile time
- **High Performance**: 2-5x faster than pandas for most operations
- **Low Memory Footprint**: Column-oriented storage with <2x overhead
- **Native Concurrency**: Built-in parallel processing using goroutines
- **Production Ready**: Comprehensive error handling and enterprise reliability
- **Easy Deployment**: Single binary with no runtime dependencies

### Design Highlights

- Column-oriented storage for cache efficiency
- Lazy evaluation with query optimization
- Zero-copy operations where possible
- Arrow-compatible memory layout
- SIMD-friendly data structures
- Immutable operations by default

## Documentation

Comprehensive documentation is available in the [`docs/`](docs/) directory:

### Core Documentation

1. **[System Design](docs/01-system-design.md)** - Architecture and core components
2. **[API Reference](docs/02-api-reference.md)** - Complete API documentation
3. **[Performance & Optimization](docs/03-performance-optimization.md)** - Performance characteristics
4. **[Advanced Operations](docs/04-advanced-operations.md)** - Joins, GroupBy, pivots, windows
5. **[Comparison with Pandas](docs/05-comparison-with-pandas.md)** - Feature comparison and trade-offs

### Getting Started

6. **[Getting Started Guide](docs/06-getting-started.md)** - Installation and basic usage
7. **[Data Types](docs/07-data-types.md)** - Type system and data types
8. **[Testing Strategy](docs/08-testing-strategy.md)** - Testing and quality assurance
9. **[Roadmap](docs/09-roadmap.md)** - Future plans and timeline

## Quick Example

```go
package main

import (
    "fmt"
    "github.com/yesoreyeram/goframes"
)

func main() {
    // Create DataFrame
    df, _ := goframes.New(map[string]interface{}{
        "name":   []string{"Alice", "Bob", "Charlie"},
        "age":    []int{25, 30, 35},
        "salary": []float64{75000, 85000, 95000},
    })

    // Filter and aggregate
    result := df.
        Filter(func(row Row) bool {
            return row["age"].(int) > 25
        }).
        GroupBy("department").
        Agg(map[string]AggFunc{
            "salary": Mean,
        })

    fmt.Println(result)
}
```

## Status

**Current Phase**: Design and Specification

This repository currently contains comprehensive design documentation for GoFrames. Implementation will follow the roadmap outlined in the documentation.

### Development Roadmap

- **Phase 1** (v0.1.0 - v0.5.0): Core foundation - 6 months
- **Phase 2** (v0.6.0 - v1.0.0): Advanced features - 6 months  
- **Phase 3** (v1.1.0 - v2.0.0): Performance and scale - 6 months
- **Phase 4** (v2.1.0+): Ecosystem and extensions - 6 months

See [Roadmap](docs/09-roadmap.md) for detailed timeline and features.

## Why GoFrames?

### When to Use GoFrames

✅ Production ETL pipelines  
✅ Real-time analytics services  
✅ High-throughput data processing  
✅ Microservices with data manipulation  
✅ Cloud-native applications  
✅ Performance-critical workloads  

### When to Use Alternatives

❌ Interactive data exploration (use Pandas)  
❌ Machine learning pipelines (use Pandas + scikit-learn)  
❌ Existing Python ecosystem integration  
❌ Rapid prototyping with notebooks  

## Performance Targets

| Operation | GoFrames | Pandas | Speedup |
|-----------|----------|--------|---------|
| CSV Read (1M rows) | 1-2s | 3-5s | 2.5-3x |
| Filter | 50ms | 180ms | 3.6x |
| GroupBy Agg | 120ms | 420ms | 3.5x |
| Join (100K rows) | 65ms | 240ms | 3.7x |
| Memory Usage | 1.5-2x | 3-4x | 50% less |

## Architecture

```
┌─────────────────────────────────────────┐
│         Application Layer                │
├─────────────────────────────────────────┤
│  API (DataFrame, Series, GroupBy, IO)   │
├─────────────────────────────────────────┤
│  Core Engine (Storage, Type, Index)     │
├─────────────────────────────────────────┤
│  Compute (Vectorized, Parallel, Agg)    │
├─────────────────────────────────────────┤
│  Storage (Memory, Buffer, Compression)  │
└─────────────────────────────────────────┘
```

## Contributing

We welcome contributions! Whether it's:

- Documentation improvements
- Design feedback
- Feature suggestions
- Bug reports
- Implementation (once we start coding)

Please see our [Roadmap](docs/09-roadmap.md) for information on how to contribute.

## License

TBD - Will be open source

## Acknowledgments

GoFrames design is inspired by:

- **Pandas**: The gold standard for DataFrame operations
- **Apache Arrow**: Cross-language columnar memory format
- **Polars**: High-performance DataFrame library
- **DuckDB**: Analytical query engine

## Contact

- **Repository**: [github.com/yesoreyeram/goframes](https://github.com/yesoreyeram/goframes)
- **Issues**: For bug reports and feature requests
- **Discussions**: For questions and design discussions

---

**Note**: This project is currently in the design phase. The documentation represents the comprehensive specification for GoFrames. Star ⭐ the repository to follow development progress!
