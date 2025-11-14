# Python AST to EMF Metamodel

Metamodel-based framework for static analysis of Python code using OCL.

Transforms Python Abstract Syntax Trees into EMF model instances, enabling constraint-based static analysis using Eclipse's OCL implementation.
Provides infrastructure to define custom OCL constraints for Python code analysis.

Uses Eclipse OCL to leverage:
- Standardized constraint language (OMG OCL 2.4 specification)
- Mature evaluation engine with debugging support
- Integration with EMF validation framework
- Existing tooling ecosystem

Metamodel maintains 1:1 structural correspondence with Python's `ast` module.

## Architecture

### Components

**Ecore Metamodel** (`dk.ingi.emf.python.ast.ecore`)
- Complete mapping of Python's `ast` module structure to EMF/Ecore
- Covers Python 3.10+ grammar including pattern matching (PEP 634)
- Preserves structural correspondence through EAnnotations

**Acceleo Template** (`generate.mtl`)
- Generates Python AST visitor from Ecore metamodel
- Produces `visitor.py` with full type hints
- Maintains synchronization between metamodel and implementation

**AST Visitor** (`visitor.py`)
- Traverses Python AST via `ast.parse()`
- Uses JPype to instantiate EMF model elements
- Serializes models as XMI for Eclipse OCL processing

### Execution Flow

```
Python source → ast.parse() → AST nodes → Visitor → EMF instances → XMI → OCL validation
```

## Capabilities

### Supported Analysis

Syntactic constraints on AST structure:
- Control flow complexity (cyclomatic, nesting depth)
- Code structure patterns (function length, decorator count)
- Naming conventions via string operations on identifiers
- Literal value detection (magic numbers, hardcoded strings)

### Example OCL Constraints

```ocl
-- Maximum function body length
context FunctionDef
inv: self.body->size() <= 50

-- Limit nested comprehension depth  
context ListComp
inv: self.generators->size() <= 2

-- Require type annotations on function arguments
context FunctionDef
inv: self.args.args->forAll(a | a.annotation <> null)
```

### Limitations

**No semantic analysis** in current implementation:
- Name resolution (which definition a Name refers to)
- Scope analysis (local/global/nonlocal)
- Type inference or checking
- Dataflow analysis (def-use chains, liveness)
- Call graph construction

These require symbol table construction, not provided in base metamodel.

## Extension

### Adding Semantic Analysis

Extend metamodel with resolution information:

```xml
<eClassifiers xsi:type="ecore:EClass" name="SymbolTable">
  <eStructuralFeatures name="symbols" eType="#//Symbol" 
                       upperBound="-1" containment="true"/>
  <eStructuralFeatures name="parent" eType="#//SymbolTable"/>
</eClassifiers>

<eClassifiers xsi:type="ecore:EClass" name="Symbol">
  <eStructuralFeatures name="name" eType="ecore:EDataType#//EString"/>
  <eStructuralFeatures name="definition" eType="#//AbstractStmt"/>
  <eStructuralFeatures name="references" eType="#//Name" upperBound="-1"/>
</eClassifiers>
```

Populate via second visitor pass using Python's `symtable` module or custom resolver.

## Usage

### Prerequisites

- Python 3.10+
- JPype1
- Eclipse with EMF/OCL plugins
- Java classpath including:
  - `org.eclipse.emf.ecore`
  - `org.eclipse.emf.common`
  - `org.eclipse.emf.ecore.xmi`

### Running

```bash
python3 visitor.py  # Parses self, outputs models/visitor.ast
```

Configure `eclipse_app` and `emf_project_dir` paths in `visitor.py`.

### OCL Validation

Load generated `.ast` files into Eclipse OCL environment. Define constraints in `.ocl` files referencing the `http://ingi.dk/emf/python` namespace.

## Technical Details

### Type Mapping

Python constants mapped to EMF via `Constant.value` (EString) and `Constant.kind`:

### XML Sanitization

Handles Python strings containing characters invalid in XML 1.0 (control characters U+0000-U+001F except tab/LF/CR) via escape sequences.

### Performance Characteristics

JPype crossing cost: O(n) where n = AST node count. Subsequent OCL evaluation runs in JVM without additional crossing overhead.

## Repository Structure

```
dk.ingi.emf.python.ast/          # EMF project
  └── model/
      └── python.ecore           # Metamodel definition

dk.ingi.emf.python.ast.api/      # Acceleo project
  └── generate.mtl               # Visitor template
```
