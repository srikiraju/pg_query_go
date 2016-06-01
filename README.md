# pg_query_go [![Build Status](https://travis-ci.org/lfittl/pg_query_go.svg)](https://travis-ci.org/lfittl/pg_query_go) [![GoDoc](https://godoc.org/github.com/lfittl/pg_query_go?status.svg)](https://godoc.org/github.com/lfittl/pg_query_go)

Go version of https://github.com/lfittl/pg_query

This Go library and its cgo extension use the actual PostgreSQL server source to parse SQL queries and return the internal PostgreSQL parse tree.

Note that the original Ruby version of this library is much more feature complete.

You can find further background to why a query's parse tree is useful here: https://pganalyze.com/blog/parse-postgresql-queries-in-ruby.html


## Installation

```
go get github.com/lfittl/pg_query_go
```

Due to compiling parts of PostgreSQL, the first time you build against this library it will take a bit longer.

Expect up to 3 minutes. You can use `go build -x` to see the progress.


## Usage

### Parsing a query into JSON

Put the following in a new Go package, after having installed pg_query as above:

```go
package main

import (
  "fmt"
  "github.com/lfittl/pg_query_go"
)

func main() {
  tree, err := pg_query.ParseToJSON("SELECT 1")
  if err != nil {
    panic(err);
  }
  fmt.Printf("%s\n", tree)
}
```

Running will output the query's parse tree as JSON:

```json
$ go run main.go
[{"SELECT": {"distinctClause": null, "intoClause": null, "targetList": [{"RESTARGET": {"name": null, "indirection": null, "val": {"A_CONST": {"val": 1, "location": 7}}, "location": 7}}], "fromClause": null, "whereClause": null, "groupClause": null, "havingClause": null, "windowClause": null, "valuesLists": null, "sortClause": null, "limitOffset": null, "limitCount": null, "lockingClause": null, "withClause": null, "op": 0, "all": false, "larg": null, "rarg": null}}]
```

### Parsing a query into Go structs

When working with the query information inside Go its recommended you use the `Parse()` method which returns Go structs:

```go
package main

import (
  "fmt"
  "reflect"
  "github.com/lfittl/pg_query_go"
  nodes "github.com/lfittl/pg_query_go/nodes"
)

func main() {
  tree, err := pg_query.Parse("SELECT 1")
  if err != nil {
    panic(err);
  }

  fmt.Printf("%s\n", reflect.DeepEqual(tree, pg_query.ParsetreeList{
    Statements: []nodes.Node{
      nodes.SelectStmt{
        TargetList: []nodes.Node{
          nodes.ResTarget{
            Val: nodes.A_Const{
              Type: "integer",
              Val: nodes.Value{
                Type: nodes.T_Integer,
                Ival: 1,
              },
              Location: 7,
            },
            Location: 7,
          },
        },
      },
    },
  }));
}
```

You can find all the node struct types in the `nodes/` directory.

## Benchmarks

As it stands, parsing has considerable overhead for complex queries, due to the use of JSON to pass structs across the C <=> Go barrier.

```
BenchmarkParseSelect1-4              	   30000	     47949 ns/op
BenchmarkParseSelect2-4              	   10000	    196460 ns/op
BenchmarkParseCreateTable-4          	    2000	    613098 ns/op
```

A good portion of this is due to JSON parsing inside Go so we can work with Go structs - just the raw parser is 10x faster:

```
BenchmarkRawParseSelect1-4           	  500000	      3796 ns/op
BenchmarkRawParseSelect2-4           	  200000	     10031 ns/op
BenchmarkRawParseCreateTable-4       	   50000	     27622 ns/op
```

Similarly, for query fingerprinting, you might want to use `pg_query.FastFingerprint` to let the C extension handle it:

```
BenchmarkFingerprintSelect1-4        	   30000	     52422 ns/op
BenchmarkFingerprintSelect2-4        	   10000	    200710 ns/op
BenchmarkFingerprintCreateTable-4    	    2000	    709827 ns/op
BenchmarkFastFingerprintSelect1-4    	  300000	      4519 ns/op
BenchmarkFastFingerprintSelect2-4    	  200000	      8255 ns/op
BenchmarkFastFingerprintCreateTable-4	  100000	     21798 ns/op
```

Normalization is already handled in the C extension, doesn't depend on JSON parsing at all, and is fast:

```
BenchmarkNormalizeSelect1-4          	 1000000	      2217 ns/op
BenchmarkNormalizeSelect2-4          	  200000	      5169 ns/op
BenchmarkNormalizeCreateTable-4      	  200000	      7923 ns/op
```

See `benchmark_test.go` for the queries.

Benchmark numbers from running on a 3.2 GHz Intel Core i5 CPU, OSX 10.11.


## Authors

- [Lukas Fittl](mailto:lukas@fittl.com)


## License

Copyright (c) 2015, Lukas Fittl <lukas@fittl.com><br>
pg_query_go is licensed under the 3-clause BSD license, see LICENSE file for details.

This project includes code derived from the [PostgreSQL project](http://www.postgresql.org/),
see LICENSE.POSTGRESQL for details.
