# Modules the Next Generation

To support the In-Service Software Update effort, we need to make reliably and
quickly bringup and teardown core primitives of FunOS -- that is, they are
attributes of the system which are functional guarantees and can serve as a
basis for other architectural features (most proximately and prominently, ISSU).

This is not to say that FunOS does not reliably boot and quiesce, but there is
currently no technological basis that guarantees the performance and
architectural behaviors of these paths, specifically with regard to dependency
handling.

Today's module system is an important step on this road and has demonstrated
that FunOS's functional units can be expressed via a statically-defined C
structure, which preserves the locality of reference between the module's code
and its metadata. But there are some areas which need to be improved upon before
we can be confident in modules as the basis for ISSU.

## Too many entry points

## Module configuration namespace is shared

## JSON (probably) not suitable for Confidential FunOS

## Modules care too much about other modules

## Mutable and immutable state are intertwined
