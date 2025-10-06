---
slug: garbage_collection
title: Garbage Collection
tags: [dotnet, gc, garbage collection, memory, performance]
---

# Garbage Collection

## Short Introduction

Garbage Collection (GC) in .NET automatically manages memory allocation and deallocation, freeing developers from manual memory management.

## Official Definition

Garbage Collection is an automatic memory management feature of .NET that periodically identifies and frees memory that is no longer being used by the application.

## Usage

```csharp
// GC is automatic, but you can interact with it
public class ResourceIntensiveClass : IDisposable
{
    private bool disposed = false;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposed)
        {
            if (disposing)
            {
                // Dispose managed resources
            }
            // Free unmanaged resources
            disposed = true;
        }
    }

    ~ResourceIntensiveClass()
    {
        Dispose(false);
    }
}

// Force garbage collection (generally not recommended)
GC.Collect();
GC.WaitForPendingFinalizers();
```

## When to Use / When Not to Use

**Generally avoid manual GC calls except:**

- Memory-intensive applications
- After large object disposal
- Performance testing scenarios

### Pros

- Automatic memory management
- Prevents memory leaks
- Optimized for application patterns

### Cons

- Non-deterministic timing
- Can cause pause times
- Less control over memory layout
