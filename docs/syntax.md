# Prerequisite Expression Language

The Prerequisite Expression Language is used to logically encode the requirements for taking a course, or for completing a degree, major, specialisation, etc.

This document provides a comprehensive guide to the language syntax, structure, and evaluation model.

## 📚 Overview

- [Basic Constructs](#basic-constructs)  
  Learn how to write simple requirements using course codes, unit counts, and wildcards.

- [General Notes on Evaluation](#general-notes-on-evaluation)  
  Understand how the evaluator matches courses against expressions and why ordering matters.

- [Advanced Constructs](#advanced-constructs)  
  Explore filters, non-consuming checks (`WEAK`), multi-clause expressions (`UNITS`), and more.

- [Examples](#examples)  
  See real-world input → output conversions to understand the language in practice.

- [EBNF Grammar](#ebnf)  
  View the full (approximate) formal grammar of the language for reference or tooling support.

---

## Basic Constructs

- `COMP1100`                                             — the default number of units of a certain course must be taken prior to now
- `~COMP1100`                                            — the default number of units of a certain course that are taken concurrently
- `6 * <COMP1100 | COMP1130>`                            — units from a group of courses (any of them can count)
- `12 * <['MATH_']>`                                     — units from a group of courses, using a wildcard (this one is 'any MATH course')
- `12 * <COMP1100 | ENGN4213 | ['MATH_'] | ['_3']>`      — units from a group of courses, mixed (any combinations are fine)
- `COMP1100 & COMP1730`                                  - 'AND' expression (the left-hand-side and right-hand-side must be satisfied)
- `COMP1100 | COMP1730`                                  - 'OR' expression (the left-hand-side or right-hand-side must be satisfied)

These can be mixed and matched using brackets and such, too. e.g.:

`COMP1100 & COMP1110 & (MATH1005 | MATH2222) & 24 * <['COMP3_'] | ['COMP4_'] | ENGN4213>`

This means: 'must take COMP1100 and COMP1110 and (either MATH1005 or MATH2222), and 24 units of (3000 or 4000-level COMP, or ENGN4213).

For instance this might be satisfied by taking:

- `COMP1100`, `COMP1110`, `MATH1005`, `COMP3500` (12 units), `ENGN4213`, `COMP4600`

Or, maybe like this!

- `COMP1100`, `COMP1110`, `MATH2222`, `COMP3600`, `COMP4600`, `COMP4670`, `COMP3900`


## Basic Wildcards

These must appear in square brackets and quotes inside a 'group' expression (the `X * < ... >` syntax above).

- `['_']`        - any course 
- `['_3']`       - any course at a given level (e.g. 'any 3000-level course')
- `['MATH_']`    - any course of a certain subject (e.g. 'any MATH course')
- `['MATH3_']`   - any course at a given level, in a certain subject (e.g. 'any 3000-level MATH course')

---


## General Notes on Evaluation

Before we get into the exact syntax, we need to look at how the planner evaluates expressions. Take the following:

`MATH1005 & 6 * <COMP1100 | ['MATH_']>`

This expression means that someone must have completed MATH1005, and then either COMP1100, or any MATH course.

Now we consider whether this expression would be satisfied if the user just took `MATH1005`. We find that it *isn't*, and this is because the evaluator **does not double count courses to requirements**. `MATH1005` could count toward its explicit mention in the expression, or against the `['MATH_']` wildcard, but it can't count for both.

Now we consider someone who has taken both `COMP1100` and `MATH1005`. We can see this is clearly satisfied: the left-hand-side of the `&` is satisfied by taking `MATH1005`, and the right-hand-side by taking `COMP1100`. 

But if we were to evaluate this 'wrongly', we can find ways for this to not be satisfied. For instance, let's evaluate the right-hand-side first. We'll match the `MATH1005` the user has taken against the `['MATH_']` wildcard (instead of `COMP1100` against its explicit requirement). Now when we get to the left-hand-side, we are stuck.

### Important Takeaway

Courses a user have taken can only be matched against a single requirement, they do not double count.

The evaluator will test all combinations of ways it can get the courses you have taken to match against the requirements, as it is sensitive to evaluation order and which courses match against which wildcards. If *any* combination can match, then the requirement is satisfied.

### But Actually...

This is a slight simplification. Courses can have various unit amounts, and they can count as long as there are units left.

e.g. if a course `COMP4500` if worth 12 units, and there is a requirement such as: `6 * <['COMP_']> & 6 * <['COMP4_']>` (i.e. 6 units of any COMP course, then 6 units of any 4000-level COMP course), it *will* match, as the 12 units can be split across both requirements. It will *also* match the requirement `COMP4500 & 6 * <['COMP4_']>`, as a raw course name (`COMP4500`) counts for the *default* number of units (6 in this case). If you have courses that are multi-unit, it is highly recommended to use the unit syntax (`12 * <COMP4500>`).

---

## Advanced Constructs

- `FILTER(expr) { subtree }` — filters combinations by a secondary condition  
- `WEAK(expr)` — condition that does not 'consume' courses  
- `UNITS 24 { MIN ...; MAX ...; }` — more complex rules

See full spec and examples below.

## Examples
Below is a list of real-world examples of course logic in English and their equivalent prerequisite expressions.

---

### Whitespace and Precedence

**Input (English):**

> COMP3670  
>  
> OR  
>  
> COMP1110 OR COMP1140  
> AND  
> MATH1014 OR MATH1115 OR MATH1116

**Expression:**

```
COMP3670 | ((COMP1110 | COMP1140) & (MATH1014 | MATH1115 | MATH1116))
```

---

### Basic Unit Conversion

**Input (English):**

> You must have completed 72 units toward a degree including BIOL1004.  
> Incompatible with BIOL1123.

**Expression:**

```
66 * <['_']> & BIOL1004
```

---

### Weak Check Alternative

**Same as above**, using a `WEAK` clause:

**Expression:**

```
72 * <['_']> & WEAK(BIOL1004)
```

---

### Advanced Filter Use

**Input (English):**

> Must complete 24 units of some list of courses, and 24 units from some other list of courses,  
> but between the two, you must take 18 units of COMP3xxx unless you take COMP4600.

**Expression:**

```
FILTER(18 * <['COMP3_']> | COMP4600) {
    24 * <...> & 24 * <...>
}
```

---

### Filter + Weak Completion Check

**Input (English):**

> Must have completed 96 units toward a degree which must include:  
> 30 units of courses at 2000/3000 level, Credit average in ENVS courses,  
> and Distinction in proposed research project.

**Expression:**

```
30 * <1 ['_2'] | ['_3']> & PC & WEAK(96 * <1 ['_']>)
```

---

### Use of `SUBST`

**Input (English):**

> Completion of one of the following computing majors:  
> COMS-MAJ, CSEC-MAJ, DTSC-MAJ, HCCC-MAJ

**Expression:**

```
SUBST("COMS-MAJ", "CSEC-MAJ", "DTSC-MAJ", "HCCC-MAJ")
```

---

### Grade-Based Entry

**Input (English):**

> Students who completed MATH1116 or MATH1113 with ≥60,  
> or MATH1013/MATH1014 with ≥80.

**Expression:**

```
MATH1116 >= 60 | MATH1113 >= 60 | MATH1013 >= 80 | MATH1014 >= 80
```

---

### Year-Dependent Path

**Input (English):**

> ACT Specialist Maths double major students may take this in first year  
> concurrently with MATH1115.

**Expression:**

```
(~MATH1115 & YEAR 1) | (MATH1116 >= 60 | MATH1113 >= 60 | MATH1013 >= 80 | MATH1014 >= 80)
```

---

### Degree-Specific with Level-Based Logic

**Input (English):**

> Enrolled in Bachelor of Laws (ALLB) and 5 LAWS 1000-level courses which may be concurrent,  
> OR Juris Doctor and 5 LAWS 1000/6100-level courses which may be concurrent.

**Expression:**

```
(DEG "Bachelor of Laws (ALLB)" & 30 * <['LAWS1_'] | [~'LAWS1_']>)
| (DEG "Juris Doctor (MJD)" & 30 * <['LAWS1_'] | [~'LAWS1_'] | ['LAWS61_'] | [~'LAWS61_']>)
```

---

### Concurrency Logic

**Input (English):**

> Must have completed or be concurrently enrolled in EMET8005 and ECON8013.  
> Excludes COMP1140.

**Expression:**

```
(EMET8005 | ~EMET8005) & (ECON8013 | ~ECON8013)
```

---

### Use of `UNITS` and `FILTER` Together

**Input (English):**

> 48 units, which must consist of at least 18 units of COMP3xxx courses, made up of:
>
> - 12 units from COMP3900 and COMP1720  
> - at least 12 units from COMP3540 | COMP4350 | COMP4610 | COMP4528
> - at most 12 units from COMP1710 | HUMN1001 | MUSI1110 | PHIL1008 
> - at most 24 units from ARTH2181 | ARTV2059 | COMP2120 | COMP3670 | DESN2004 | DESN2008 | DESN2010 | HUMN2001 | MGMT2009 | MUSI3309 | SCOR3001 | SOCY2038 | SOCY2166

**Expression:**

```
COMP1720 & COMP3900 & FILTER(12 * <['COMP3_']>) {
    UNITS 36 {
        MIN 12 * <COMP3540 | COMP4350 | COMP4610 | COMP4528>
        MAX 12 * <COMP1710 | HUMN1001 | MUSI1110 | PHIL1008>
        MAX 24 * <ARTH2181 | ARTV2059 | COMP2120 | COMP3670 | DESN2004 | DESN2008 | DESN2010 | HUMN2001 | MGMT2009 | MUSI3309 | SCOR3001 | SOCY2038 | SOCY2166>
    }
}
```

**Explanatory Notes**

`COMP3900` and `COMP1720` are required, and so get excluded from the `UNITS` block. Between them, they make up 12 units, so this 12 is removed
from the 48 required, to give a `UNITS 36 { ... }`. We additionally lower the 18 units of COMP3xxx down to 12 units, because we are required
to take `COMP3900`, which counts for 6 units.


## EBNF
This is *not* guaranteed to be correct, but just a sketch that may be of use.

```ebnf
(* -------------  Top-level ------------- *)
<expression>        ::= <or-expr> ;

<or-expr>           ::= <and-expr>
                      | <or-expr> "|" <and-expr> ;

<and-expr>          ::= <term>
                      | <and-expr> "&" <term> ;

(* -------------  Terms / primaries ------------- *)
<term>              ::= <parenthesised>
                      | <unit-expr>
                      | <special-expr>
                      | <course-term>
                      | <group-expr> ;

<parenthesised>     ::= "(" <expression> ")" ;

(* -------------  Courses, wildcards, groups ------------- *)
<course-term>       ::= [ "!" ] [ "~" ] <COURSE_CODE>
                      | [ "!" ] [ "~" ] "[" [ "~" ] <STRING_LITERAL> "]" ;

<group-expr>        ::= "<"
                           [ "1" ]                       (* “stop-after-first” hint *)
                           <group-item>
                           { "|" <group-item> }
                        ">"
                        [ ">=" <INTEGER> ]               (* per-course minimum grade *)
                        ;

<group-item>        ::= <course-term> ;                  (* Courses and/or wildcards *)

<unit-expr>         ::= <units-prefix> <course-or-group>
                     | <courses-default-units> ;

<units-prefix>      ::= <INTEGER> "*" ;                  (* Explicit unit count *)

<course-or-group>   ::= <course-term> | <group-expr> ;

<courses-default-units>
                    ::= <course-term> ;                  (* Implicit units = default *)

(* -------------  “UNITS n { … }” ------------- *)

<units-block>       ::= "UNITS" <INTEGER>
                        "{"
                           <minmax-clause>
                           { <minmax-clause> }
                        "}" ;

<minmax-clause>     ::= ( "MIN" | "MAX" )
                        <INTEGER> "*"
                        <group-expr> ;

(* -------------  Filters and WEAK ------------- *)
<filter-expr>       ::= "FILTER"
                        "("  <expression> ")"            (* WEAK filter test        *)
                        "{"
                            <expression>                 (* Main counted expression *)
                        "}" ;

<weak-expr>         ::= "WEAK" "(" <expression> ")" ;

(* -------------  Then / After ------------- *)
<then-after-expr>   ::= ( "THEN" | "AFTER" )
                        <COURSE_CODE>
                        [ "YEAR" <INTEGER> ]             (* fixed year, optional    *)
                        [ <STRING_LITERAL> ]             (* fixed semester, optional*)

(* -------------  Other atomic special forms ------------- *)
<grade-expr>        ::= <COURSE_CODE> ">=" <INTEGER> ;

<gpa-expr>          ::= "GPA"  ">=" <INTEGER>            (* or ×10 for halves       *)
                      | "GPA"  ">=" <INTEGER>.<INTEGER> ;

<wam-expr>          ::= "WAM"  ">=" <INTEGER> ;

<year-expr>         ::= "YEAR" <INTEGER> [ "+" ] ;

<degree-expr>       ::= "DEG" <STRING_LITERAL> ;

<permission-expr>   ::= "PC"  [ <STRING_LITERAL> ] ;

<subst-expr>        ::= "SUBST"
                        <string-list> ;

<select-expr>       ::= "SELECT"
                        <STRING_LITERAL>                 (* selector id            *)
                        <string-list> ;                  (* same list syntax       *)

<string-list>       ::= <STRING_LITERAL>
                        { "," <STRING_LITERAL> } ;

<constant>          ::= "TRUE" | "FALSE" ;

<other-expr>        ::= "OTHER" <STRING_LITERAL> ;       (* school-specific hook   *)

(* -------------  Catch-all for any primary that isn’t     *)
<special-expr>      ::= <units-block>
                     | <filter-expr>
                     | <weak-expr>
                     | <then-after-expr>
                     | <grade-expr>
                     | <gpa-expr>
                     | <wam-expr>
                     | <year-expr>
                     | <degree-expr>
                     | <permission-expr>
                     | <subst-expr>
                     | <select-expr>
                     | <constant>
                     | <other-expr> ;

(* -------------  Lexical tokens (regex sketches) -------- *)
<COURSE_CODE>       ::= /[A-Z]{4}\d{4}/ ;
<STRING_LITERAL>    ::= '"' ... '"'                       (* any printable except " *)
<INTEGER>           ::= /0|[1-9]\d*/ ;
```