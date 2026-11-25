# JSON (`encoding/json`)

## Struct tags

```go
type User struct {
    ID        int64      `json:"id"`
    Email     string     `json:"email"`
    Nickname  string     `json:"nickname,omitempty"`  // omitted if ""
    Age       int        `json:"age,string"`          // encoded as "42" (a string)
    Password  string     `json:"-"`                   // never encoded or decoded
    Dash      string     `json:"-,"`                  // key literally named "-"
    CreatedAt time.Time  `json:"created_at"`          // RFC 3339
    internal  string                                 // unexported, so always ignored
}
```

- Only **exported** fields are encoded and decoded.
- Without a tag, the key is the Go field name (`"Email"`).
- Embedded struct fields are **flattened** into the parent object.
- `[]byte` is encoded as base64. Map keys are sorted on output.

## Omitted fields

`omitempty` omits: `false`, `0`, `""`, nil pointers or interfaces, and **empty** slices, maps, or arrays (len 0).

It does **not** omit an empty struct or a zero `time.Time`:

```go
type T struct {
    When time.Time `json:"when,omitempty"`   // still prints "0001-01-01T00:00:00Z"
    Addr Address   `json:"addr,omitempty"`   // still prints {}
}
```

Go 1.24+ adds **`omitzero`**: it omits the zero value of any type, and calls `IsZero() bool` if the type has one.

```go
When time.Time `json:"when,omitzero"`   // omitted when When.IsZero()
```

## null values

**Encoding:**
- nil pointer, nil interface, nil slice, nil map → `null`
- empty slice `[]T{}` → `[]`, empty map → `{}`

**Decoding:**
- `null` into a pointer, slice, map, or interface sets it to **nil**.
- `null` into a non-pointer (`int`, `string`, struct) is a **no-op**: the field keeps whatever it had.
- A **missing** key also leaves the field untouched. Decoding into a pre-filled struct only overwrites what's present.

### Missing vs null vs zero

| Need to distinguish | Field type |
|---|---|
| nothing (zero is fine) | `int` |
| "provided" vs "not provided or null" | `*int` (nil means missing or null) |
| missing vs `null` vs value (PATCH semantics) | custom type, or `json.RawMessage` |

```go
type Optional[T any] struct {
    Set   bool   // key was present
    Null  bool   // value was null
    Value T
}

// Only called when the key is present (including for null)
func (o *Optional[T]) UnmarshalJSON(b []byte) error {
    o.Set = true
    if string(b) == "null" {
        o.Null = true
        return nil
    }
    return json.Unmarshal(b, &o.Value)
}

type PatchUser struct {
    Nickname Optional[string] `json:"nickname"`
}
```

## Input validation

`encoding/json` is **lenient** by default. Here's what it allows silently:

| Default behavior | Fix |
|---|---|
| Unknown fields are ignored | `dec.DisallowUnknownFields()` |
| Keys match fields **case-insensitively** (`"EMAIL"` fills `Email`) | Validate afterwards, or use json/v2 (see below) |
| Missing required fields stay zero | Validate explicitly after decoding |
| Trailing data after the object (with `Decoder`) | Decode again and require `io.EOF` |
| Huge request bodies | `http.MaxBytesReader` |
| Numbers into `any` become `float64` (precision loss above 2^53) | `dec.UseNumber()` or decode into a typed field |

```go
type CreateUser struct {
    Email string `json:"email"`
    Age   *int   `json:"age"`
}

func (c CreateUser) Validate() error {
    var errs []error
    if c.Email == "" || !strings.Contains(c.Email, "@") {
        errs = append(errs, errors.New("email: required and must be valid"))
    }
    if c.Age != nil && (*c.Age < 0 || *c.Age > 150) {
        errs = append(errs, errors.New("age: out of range"))
    }
    return errors.Join(errs...)   // nil if there are no errors
}

func decodeCreateUser(w http.ResponseWriter, r *http.Request) (CreateUser, error) {
    var in CreateUser
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1 MiB

    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()

    if err := dec.Decode(&in); err != nil {
        return in, fmt.Errorf("decode body: %w", err)
    }
    if err := dec.Decode(&struct{}{}); err != io.EOF {
        return in, errors.New("body must contain a single JSON object")
    }
    return in, in.Validate()
}
```

Useful decode error types for friendly messages: `*json.SyntaxError` (has `.Offset`), `*json.UnmarshalTypeError` (has `.Field`, `.Value`), and `*http.MaxBytesError`.

## Custom (un)marshalling

```go
type Status int

// Value receiver: works for both Status and *Status
func (s Status) MarshalJSON() ([]byte, error) {
    return json.Marshal(s.String())
}

// Pointer receiver: it has to modify s
func (s *Status) UnmarshalJSON(b []byte) error {
    var str string
    if err := json.Unmarshal(b, &str); err != nil { return err }
    v, ok := parseStatus(str)
    if !ok { return fmt.Errorf("invalid status %q", str) }
    *s = v
    return nil
}
```

If you only need string conversion, `encoding.TextMarshaler` / `TextUnmarshaler` also works, and it works for map keys too.

## Other tools

```go
json.RawMessage          // defer decoding a sub-tree (polymorphic payloads)
enc.SetIndent("", "  ")  // pretty-print; json.MarshalIndent also exists
enc.SetEscapeHTML(false) // by default <, >, & are escaped as < etc.
```

## json/v2 (experimental)

Go 1.25 ships `encoding/json/v2` behind `GOEXPERIMENT=jsonv2`. Its notable defaults: **case-sensitive** matching, duplicate keys rejected, and nil slices/maps encoded as `[]`/`{}`. It's worth knowing it exists, but v1 is still what most code uses.

## Be ready to explain

- `omitempty` vs `omitzero`, and why a zero `time.Time` isn't omitted by `omitempty`.
- How you'd tell "field missing", "field null", and "field zero" apart.
- Everything `encoding/json` lets through by default, and how you'd lock down an API handler.
