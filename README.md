# UniFFI Bug Repro

## Bug

UniFFI tends to create duplicate declarations for custom types that use the new `bytes` UDL type AND appear in function parameters.

In example for kotlin the types that are emitted are both `ByteArray` (correct) and `List<UByte>` (incorrect) - same goes for Swift's `Data` / `[UInt8]`, Python's `bytes` / `typing.List[int]`.

This leads me to think that the issue is not in the bindings generation but above that, probably in type disambiguation.


## Steps to repro

```
$ cargo build --release
$ for lang in python swift kotlin; do cargo run --features uniffi/cli --bin uniffi-bindgen -- generate --language $lang --out-dir ./bindings/$lang --library target/release/libuniffi_custom_bug.so
```

Then check the bindings folder for faulty types declarations (search for `CustomType`).

### Examples of double declarations

<details>
    <summary>#### Kotlin</summary>

```kotlin
// ./bindings/kit/uniffi/UniffiCustomBug/UniffiCustomBug.kt:477

/**
 * Typealias from the type name used in the UDL file to the builtin type.  This
 * is needed because the UDL type name is used in function/method signatures.
 * It's also what we have an external type that references a custom type.
 */
public typealias CustomType = ByteArray
public typealias FfiConverterTypeCustomType = FfiConverterByteArray

/**
 * Typealias from the type name used in the UDL file to the builtin type.  This
 * is needed because the UDL type name is used in function/method signatures.
 * It's also what we have an external type that references a custom type.
 */
public typealias CustomType = List<UByte>
public typealias FfiConverterTypeCustomType = FfiConverterSequenceUByte
```

</details>

<details>
    <summary>#### Swift</summary>

```swift
// ./bindings/swift/UniffiCustomBug.swift:349


/**
 * Typealias from the type name used in the UDL file to the builtin type.  This
 * is needed because the UDL type name is used in function/method signatures.
 */
public typealias CustomType = Data
public struct FfiConverterTypeCustomType: FfiConverter {
    public static func read(from buf: inout (data: Data, offset: Data.Index)) throws -> CustomType {
        return try FfiConverterData.read(from: &buf)
    }

    public static func write(_ value: CustomType, into buf: inout [UInt8]) {
        return FfiConverterData.write(value, into: &buf)
    }

    public static func lift(_ value: RustBuffer) throws -> CustomType {
        return try FfiConverterData.lift(value)
    }

    public static func lower(_ value: CustomType) -> RustBuffer {
        return FfiConverterData.lower(value)
    }
}


/**
 * Typealias from the type name used in the UDL file to the builtin type.  This
 * is needed because the UDL type name is used in function/method signatures.
 */
public typealias CustomType = [UInt8]
public struct FfiConverterTypeCustomType: FfiConverter {
    public static func read(from buf: inout (data: Data, offset: Data.Index)) throws -> CustomType {
        return try FfiConverterSequenceUInt8.read(from: &buf)
    }

    public static func write(_ value: CustomType, into buf: inout [UInt8]) {
        return FfiConverterSequenceUInt8.write(value, into: &buf)
    }

    public static func lift(_ value: RustBuffer) throws -> CustomType {
        return try FfiConverterSequenceUInt8.lift(value)
    }

    public static func lower(_ value: CustomType) -> RustBuffer {
        return FfiConverterSequenceUInt8.lower(value)
    }
}

```

</details>

<details>
    <summary>#### Python</summary>

```python
# ./bindings/python/UniffiCustomBug.py:621


# Type alias
CustomType = bytes

class FfiConverterTypeCustomType:
    @staticmethod
    def write(value, buf):
        FfiConverterBytes.write(value, buf)

    @staticmethod
    def read(buf):
        return FfiConverterBytes.read(buf)

    @staticmethod
    def lift(value):
        return FfiConverterBytes.lift(value)

    @staticmethod
    def lower(value):
        return FfiConverterBytes.lower(value)


# Type alias
CustomType = typing.List[int]

class FfiConverterTypeCustomType:
    @staticmethod
    def write(value, buf):
        FfiConverterSequenceUInt8.write(value, buf)

    @staticmethod
    def read(buf):
        return FfiConverterSequenceUInt8.read(buf)

    @staticmethod
    def lift(value):
        return FfiConverterSequenceUInt8.lift(value)

    @staticmethod
    def lower(value):
        return FfiConverterSequenceUInt8.lower(value)


```

</details>


