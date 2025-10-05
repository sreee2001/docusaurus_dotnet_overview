### Common Language Infrastructure (CLI)

#### Short Introduction

CLI is the open specification that describes the executable code and runtime environment that allows multiple high-level languages to be used on different computer platforms.

#### Official Definition

The Common Language Infrastructure (CLI) is an open specification developed by Microsoft that describes executable code and a runtime environment that allows multiple high-level languages to be used on different computer platforms without being rewritten for specific architectures.

#### Usage

Key components:

- Common Type System (CTS)
- Common Language Specification (CLS)
- Virtual Execution System (VES)
- Metadata system

#### Use Cases

- Cross-language interoperability
- Platform independence
- Code sharing between different .NET languages

#### Sample Usage

```csharp
// C# code
public class Calculator
{
    public int Add(int a, int b) => a + b;
}
```

```vb
' VB.NET code using the same CLI
Public Class Calculator
    Public Function Add(a As Integer, b As Integer) As Integer
        Return a + b
    End Function
End Class
```

---
