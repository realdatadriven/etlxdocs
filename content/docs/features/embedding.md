---
weight: 810
title: "Embedding in GO"
description: "Embedding ETLX in Go applications for seamless integration."
icon: code-branch
tags: ["features"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Embedding in Go

To embed the ETL framework in a Go application, you can use the `etlx` package and call `ConfigFromMDText` and `RunETL`. Example (from README):

```go
package main

import (
    "fmt"
    "time"
    "github.com/realdatadriven/etlx"
)

func main() {
    etl := &etlx.ETLX{}

    // Load configuration from Markdown text
    err := etl.ConfigFromMDText(`# Your Markdown config here`)
    if err != nil {
        fmt.Printf("Error loading config: %v\n", err)
        return
    }

    // Prepare date reference
    dateRef := []time.Time{time.Now().AddDate(0, 0, -1)}

    // Define additional options
    options := map[string]any{
        "only":  []string{"sales"},
        "steps": []string{"extract", "load"},
    }

    // Run ETL process
    logs, err := etl.RunETL(dateRef, nil, options)
    if err != nil {
        fmt.Printf("Error running ETL: %v\n", err)
        return
    }

    // Print logs
    for _, log := range logs {
        fmt.Printf("Log: %+v\n", log)
    }
}
```
This code snippet demonstrates how to set up and run an ETL process using the ETLX framework within a Go application. You can customize the configuration and options as needed for your specific use case.

>The binary `etlx` (see realdatadriven/etlx/cmd/main.go) itself also uses this embedding approach internally to run ETL processes defined in Markdown files .
