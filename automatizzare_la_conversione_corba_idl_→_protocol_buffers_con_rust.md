# Automatizzare la conversione CORBA IDL → Protocol Buffers con un'applicazione Rust

## 1. Obiettivo
Questo documento descrive come progettare e implementare una **applicazione Rust** che automatizzi la conversione di file **CORBA IDL** in definizioni **Protocol Buffers**, preservando:
- firme delle operazioni
- valori di ritorno
- parametri `in / out / inout`
- eccezioni CORBA

L'obiettivo è realizzare un **tool CLI** ripetibile, estendibile e integrabile in pipeline CI.

---

## 2. Architettura generale del tool

```
+-------------------+
| File IDL (.idl)   |
+---------+---------+
          |
          v
+-------------------+
| Parser IDL        |  (AST)
+---------+---------+
          |
          v
+-------------------+
| Modello intermedio|  (IR)
+---------+---------+
          |
          v
+-------------------+
| Code Generator    |
| (Protocol Buffers)|
+---------+---------+
          |
          v
+-------------------+
| File .proto       |
+-------------------+
```

### Componenti chiave
- **Parser**: trasforma IDL in AST
- **IR (Intermediate Representation)**: modello neutro
- **Generator**: emette `.proto`

---

## 3. Scelta tecnologica in Rust

### 3.1 Parsing CORBA IDL
Opzioni realistiche:

1. **tree-sitter-idl** (consigliato)
   - parsing robusto
   - gestione errori
   - già familiare se usi tree-sitter in altri tool

2. **Parser custom (PEG / nom)**
   - alto costo
   - sconsigliato per IDL completo

**Scelta raccomandata:** `tree-sitter`.

### 3.2 Crate principali

```toml
[dependencies]
tree-sitter = "0.22"
tree-sitter-corba-idl = "*"   # se disponibile o fork interno
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
```

---

## 4. Modellazione dell'AST → IR

### 4.1 Perché un IR
L'IDL AST è troppo sintattico. Un **IR semantico** semplifica:
- mapping verso Protobuf
- supporto futuro ad altri target (OpenAPI, Thrift, ecc.)

### 4.2 Esempio di IR in Rust

```rust
#[derive(Debug)]
pub struct Interface {
    pub name: String,
    pub operations: Vec<Operation>,
}

#[derive(Debug)]
pub struct Operation {
    pub name: String,
    pub params: Vec<Parameter>,
    pub return_type: Option<Type>,
    pub exceptions: Vec<Exception>,
}

#[derive(Debug)]
pub struct Parameter {
    pub name: String,
    pub ty: Type,
    pub direction: ParamDirection,
}

#[derive(Debug)]
pub enum ParamDirection {
    In,
    Out,
    InOut,
}
```

---

## 5. Parsing IDL con tree-sitter

### 5.1 Pipeline di parsing

```rust
let mut parser = Parser::new();
parser.set_language(tree_sitter_corba_idl::language())?;

let tree = parser.parse(idl_source, None)
    .ok_or_else(|| anyhow!("Parse error"))?;
```

### 5.2 Visitor sull'AST

- attraversamento dei nodi `interface`
- estrazione di:
  - operazioni
  - parametri
  - tipo di ritorno
  - clausola `raises`

Il visitor popola l'IR.

---

## 6. Mapping IR → Protocol Buffers

### 6.1 Strategia

| IR | Protobuf |
|----|----------|
| Interface | `service` |
| Operation | `rpc` |
| Param `in` | Request field |
| Param `out` | Response field |
| Return type | Response field `result` |
| Exceptions | `oneof error` |

---

## 7. Generazione dei messaggi Protobuf

### 7.1 Request

```rust
fn gen_request(op: &Operation) -> String {
    // genera message <OpName>Request
}
```

Include:
- tutti i parametri `in`
- parte `inout`

### 7.2 Response

```rust
fn gen_response(op: &Operation) -> String {
    // oneof { result | error }
}
```

Contiene:
- `result` se `return_type` != void
- campi `out` / `inout`
- wrapper errori

---

## 8. Generazione delle eccezioni

### 8.1 Exception → message

```proto
message InsufficientFunds {
  double balance = 1;
}
```

### 8.2 Aggregazione errori

```proto
message WithdrawError {
  oneof error {
    InsufficientFunds insufficient_funds = 1;
    AccountNotFound account_not_found = 2;
  }
}
```

---

## 9. Generazione del service

```proto
service AccountService {
  rpc Withdraw(WithdrawRequest) returns (WithdrawResponse);
}
```

Il nome RPC deriva direttamente dall'IDL.

---

## 10. CLI del tool

### 10.1 Interfaccia utente

```bash
idl2proto \
  --input account.idl \
  --output account.proto \
  --package banking.account
```

### 10.2 Clap

```rust
#[derive(Parser)]
struct Cli {
    #[arg(short, long)]
    input: PathBuf,

    #[arg(short, long)]
    output: PathBuf,

    #[arg(long)]
    package: Option<String>,
}
```

---

## 11. Estensioni avanzate

### 11.1 Supporto multi-file IDL
- risoluzione `#include`
- symbol table globale

### 11.2 Output multipli
- `.proto`
- JSON (IR)
- documentazione Markdown

### 11.3 Validazione
- collisioni di nomi
- numeri di campo Protobuf
- compatibilità backward

---

## 12. Testing

- Golden tests (`IDL → .proto`)
- Snapshot testing
- Fuzzing del parser IDL

---

## 13. Conclusione

Un tool Rust basato su **tree-sitter + IR + code generation** consente di automatizzare in modo affidabile la conversione da CORBA IDL a Protocol Buffers, mantenendo la semantica originale e producendo contratti moderni e versionabili.

L'approccio è scalabile, testabile e allineato a best practice industriali per la migrazione di middleware legacy.

