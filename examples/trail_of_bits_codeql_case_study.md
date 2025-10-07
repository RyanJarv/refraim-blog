# Taming 2,500 Compiler Warnings with CodeQL, an OpenVPN2 Case Study

**Paweł Płatek, Jay Little**
**September 25, 2025**
**codeql, c/c++, static-analysis**

## Page content
* Why compiler warnings aren’t enough
* When conversions matter for security
* Building a practical CodeQL query
  * Step 0: Learn from existing CodeQL queries
  * Step 1: Find all problematic conversions (7,000+ findings)
  * Step 2: Eliminate provably safe constants (1,017 findings)
  * Step 3: Apply range analysis (435 findings)
  * Step 4: Model codebase-specific knowledge (254 findings)
  * Step 5: Focus on user-controlled inputs (20 findings)
* Securing your code against silent failures

Why are implicit integer conversions a problem in C?

```c
if (-7 > sizeof(int)) {
    puts("That's why.");
}
```

During our security review of OpenVPN2, we faced a daunting challenge: which of the about 2,500 implicit conversions compiler warnings could actually lead to a vulnerability? To answer this, we created a new CodeQL query that reduced the number of flagged implicit conversions to just 20. Here is how we built the query, what we learned, and how you can run the queries on your code. Our query is available on GitHub, and you can dig deeper into the details in our full case study paper.

## WHY COMPILER WARNINGS AREN’T ENOUGH

Modern compilers detect implicit conversions with flags like `-Wconversion`, but can generate a massive number of warnings because they do not distinguish between which are benign and which are dangerous for security purposes. When we compiled OpenVPN2 with conversion detection flags, we found thousands of warnings:

*   GCC 14.2.0: 2,698 reported warnings with `-Wconversion -Wsign-conversion -Wsign-compare`
*   Clang 19.1.7: 2,422 reported warnings with `-Wsign-compare -Wsign-conversion -Wimplicit-int-conversion -Wshorten-64-to-32`

Manual review of 2,500+ findings is impractical, and most warnings highlight benign conversions. The challenge isn’t identifying conversions—it’s determining which ones introduce security vulnerabilities.

## WHEN CONVERSIONS MATTER FOR SECURITY

C’s relaxed type system allows for implicit conversions, which is when the compiler automatically changes the type of a variable to make code compile. Not all conversions are problematic, but this behavior creates space for vulnerabilities. One problematic case is when the result of the conversion is used to alter data. To better understand the ways in which data alteration can be problematic, we have broken it down into three categories: truncation, reinterpretation, and widening.

Here is a concise example of each (for more details, check out the full paper):

```c
unsigned int x = 0x80000000;

unsigned char a = x;  // truncation
int b = x;  // reinterpretation
uint64_t c = b;  // widening
```

The examples above were all altered via the same type of conversion: conversion as if by assignment. There are two other types of conversions that C programmers often encounter.

Usual arithmetic conversion occurs when variables of different types are operated on and reconciled:

```c
unsigned short header_size = 0x13;
int offset = 0x37;
return header_size + offset;  // usual arithmetic conversion
```

Integer promotions happen when unary bitwise, arithmetic, or shift operations happen on a single variable:

```c
uint8_t val = 0x13;
int val2 = (~val) >> 3;  // integer promotion
```

By combining the conversion types with the data alteration types mentioned above, we can create a table to clarify which implicit conversions we should further analyze for possible security issues.

| | Truncation | Reinterpretation | Widening |
|---|---|---|---|
| **As if by assignment** | Possible | Possible | Possible |
| **Integer promotions** | Not possible | Not possible | Possible |
| **Usual arithmetic conversions** | Not possible | Possible | Possible |

## BUILDING A PRACTICAL CODEQL QUERY

Back to our security review of OpenVPN2, where we encountered more than 2,500 compiler warnings flagging implicit conversions. Rather than manually reviewing the thousands of warnings, we built a CodeQL query through iterative refinement. Each step improved the query to eliminate classes of false positives while preserving the semantics we cared about for security purposes.

### STEP 0: LEARN FROM EXISTING CODEQL QUERIES

Before writing a new query, we wanted to review existing queries that may be relevant or useful. We found three queries, but like Goldilocks, we found that none were a match for what we wanted. Each was either too noisy or checked only a subset of conversions.

*   `cpp/conversion-changes-sign`: 988 findings. It detects only implicit unsigned-to-signed integer conversions and only filters out conversions with const values.
*   `cpp/jsf/av-rule-180`: 6,750 findings. It detects only up to 32-bit types and does not report widening-related issues.
*   `cpp/sign-conversion-pointer-arithmetic`: 1 finding. It checks only when type conversions are used for pointer arithmetic. It also covers explicit conversions.

### STEP 1: FIND ALL PROBLEMATIC CONVERSIONS (7,000+ FINDINGS)

Our initial query found every implicit integer conversion and returned over 7,000 results in the OpenVPN2 codebase:

```ql
import cpp

from IntegralConversion cast, IntegralType fromType, IntegralType toType
where
    cast.isImplicit()
    and fromType = cast.getExpr().getExplicitlyConverted().getUnspecifiedType()
    and toType = cast.getUnspecifiedType()
    and fromType != toType
    and not toType instanceof BoolType

select cast, "Implicit cast from " + fromType + " to " + toType
```

This was expectedly broad, so we then updated it to filter the cases we were actually interested in, cutting the results to 5,725:

```ql
and (
    // truncation
    fromType.getSize() > toType.getSize()
    or
    // reinterpretation
    (
        fromType.getSize() = toType.getSize()
        and
        (
            (fromType.isUnsigned() and toType.isSigned())
            or
            (fromType.isSigned() and toType.isUnsigned())
        )
    )
    or
    // widening
    (
        fromType.getSize() < toType.getSize()
        and
        (
            (fromType.isSigned() and toType.isUnsigned())
            or
            // unsafe promotion
            exists(ComplementExpr complement |
                complement.getOperand().getConversion*() = cast
            )
        )
    )
)

and not (
    // skip conversions in arithmetic operations
    fromType.getSize() <= toType.getSize() // should always hold
    and exists(BinaryArithmeticOperation arithmetic |
        (arithmetic instanceof AddExpr or arithmetic instanceof SubExpr or arithmetic instanceof MulExpr)
        and arithmetic.getAnOperand().getConversion*() = cast
    )
```

### STEP 2: ELIMINATE PROVABLY SAFE CONSTANTS (1,017 FINDINGS)

Many conversions involve compile-time constants that will never cause problems:

```c
uint32_t safe_value = 42;
uint16_t result = safe_value;  // safe conversion
```

We created a new predicate to model safe ranges of constant values:

```ql
import semmle.code.cpp.rangeanalysis.RangeAnalysisUtils

predicate isSafeConstant(Expr cast, IntegralType toType) {
    exists(float knownValue |
        knownValue = cast.getValue().toFloat()
        and knownValue <= typeUpperBound(toType)
        and knownValue >= typeLowerBound(toType)
    )
}
```

This filter reduced the findings to 1,017 by checking that constants are within the expected range and filtering safe equality checks.

### STEP 3: APPLY RANGE ANALYSIS (435 FINDINGS)

CodeQL’s range analysis can determine the possible minimum and maximum values of variables. We progressively applied different types of range analysis:

*   `SimpleRangeAnalysis` reduced the query to 913 results.
*   `ExtendedRangeAnalysis`’s classes combined with our own newly created `ConstantBitwiseOrExprRange` class reduced the results to 886.

CodeQL’s `SimpleRangeAnalysis` is intraprocedural, but we had ideas for handling some simple interprocedural cases, such as this one:

```c
static inline bool
is_ping_msg(const struct buffer *buf)
{
    // the only call to buf_string_match
    return buf_string_match(buf, ping_string, 16);
}

static inline bool
buf_string_match(const struct buffer *src, const void *match, int size)
{
    if (size != src->len)
    {
        return false;
    }
    // size is always safely converted
    return memcmp(BPTR(src), match, size) == 0;
}
```

By extending the `SimpleRangeAnalysisDefinition` class to constrain function arguments, we reduced the findings to 575!

By using IR-based `RangeAnalysis`, we further reduced the findings to 435, but it significantly increased the runtime of the query. See the paper for more specific details.

### STEP 4: MODEL CODEBASE-SPECIFIC KNOWLEDGE (254 FINDINGS)

We created models for functions in OpenVPN2, the C standard library, and OpenSSL that bound their return values. These simple additions further improved the range analysis by eliminating findings related to known-safe functions. This domain-specific knowledge reduced our findings to 254.

Below are two examples of these new function models:

```ql
private class BufLenFunc extends SimpleRangeAnalysisExpr, FunctionCall {
  BufLenFunc() {
    this.getTarget()
        .getName()
        .matches([
            "buf_len", "buf_reverse_capacity", "buf_forward_capacity", "buf_forward_capacity_total"
          ])
  }

  override float getLowerBounds() { result = 0 }

  override float getUpperBounds() { result = typeUpperBound(this.getExpectedReturnType()) }

  override predicate dependsOnChild(Expr child) { none() }
}

private class OpenSSLFunc extends SimpleRangeAnalysisExpr, FunctionCall {
  OpenSSLFunc() {
    this.getTarget()
        .getName()
        .matches([
            "EVP_CIPHER_get_block_size", "cipher_ctx_block_size", "EVP_CIPHER_CTX_get_block_size",
            "EVP_CIPHER_block_size", "HMAC_size", "hmac_ctx_size", "EVP_MAC_CTX_get_mac_size",
            "EVP_CIPHER_CTX_mode", "EVP_CIPHER_CTX_get_mode", "EVP_CIPHER_iv_length",
            "cipher_ctx_iv_length", "EVP_CIPHER_key_length", "EVP_MD_size", "EVP_MD_get_size",
            "cipher_kt_iv_size", "cipher_kt_block_size", "EVP_PKEY_get_size", "EVP_PKEY_get_bits",
            "EVP_PKEY_get_security_bits"
          ])
  }

  override float getLowerBounds() { result = 0 }

  override float getUpperBounds() { result = 32768 }

  override predicate dependsOnChild(Expr child) { none() }
```

### STEP 5: FOCUS ON USER-CONTROLLED INPUTS (20 FINDINGS)

Finally, we used taint tracking and sources provided by the `FlowSource` classes to identify conversions involving user-controlled data, the most likely source of exploitable vulnerabilities. This final filter brought us down to just 20 high-priority cases for manual review.

After analyzing these remaining cases, we found that none were exploitable in OpenVPN2’s context. No vulnerabilities, but it’s a win anyway: we checked all of OpenVPN2’s implicit conversions, we saved a lot of manual-review time, and now we have a reusable CodeQL query for anyone to use on their C codebases.

## SECURING YOUR CODE AGAINST SILENT FAILURES

Take these steps to detect problematic implicit conversions in your C codebase:

1.  Run our CodeQL query against your C codebase to eliminate the most urgent issues.
2.  Add our query to your build system to continuously look for implicit conversion bugs.
3.  Establish coding standards that minimize or eliminate implicit conversions.
4.  Document and justify nonobvious explicit conversions.
5.  Once your project is mature enough, turn on the `-Wconversion -Wsign-compare` compiler flags and treat related warnings as errors.

Implicit conversions represent a fundamental mismatch between developer intent and compiler behavior. While C’s permissive approach may seem convenient, it creates opportunities for subtle security vulnerabilities that are difficult to spot in code review.

The key insight from our OpenVPN2 analysis is that most implicit conversions are benign, and identifying the subset of dangerous conversions requires sophisticated analysis. By combining compiler warnings with targeted static analysis, and consistent coding practices, you can significantly reduce your exposure to these invisible security flaws.
