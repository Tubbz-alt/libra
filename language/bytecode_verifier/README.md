---
id: bytecode-verifier
title: Bytecode Verifier
custom_edit_url: https://github.com/libra/libra/edit/master/language/bytecode_verifier/README.md
---

### Bytecode Verifier: Checking Safety of Stack Usage, Types, Resources, and References

The body of each function in a compiled module is verified separately while trusting the correctness of function signatures in the module.  Checking that each function signature matches its definition is a separate responsibility.  The body of a function is a sequence of bytecode instructions.  This instruction sequence is checked in several phases described below.

## CFG Construction

A control-flow graph is constructed by decomposing the instruction sequence into a collection of basic blocks.  Each basic block contains a contiguous sequence of instructions; the set of all instructions is partitioned among the blocks.  Each block ends with a branch or return instruction.  The decomposition into blocks guarantees that branch targets land only at the beginning of some block.  The decomposition also attempts to ensure that the generated blocks are maximal.  However, the soundness of the analysis does not depend on maximality.

## Stack Safety

The execution of a block happens in the context of an array of local variables and a stack.  The parameters of the function are a prefix of the array of local variables.  Passing arguments and return values across function calls is done via the stack.  When a function starts executing, its arguments are already loaded into its parameters.  Suppose the stack height is *n* when a function starts executing; then valid bytecode must enforce the invariant that when execution lands at the beginning of a basic block, the stack height is *n*.  Furthermore, at a return instruction, the stack height must be *n*+*k* where *k*, s.t. *k*>=0 is the number of return values.  The first phase of the analysis checks that this invariant is maintained by analyzing each block separately, calculating the effect of each instruction in the block on the stack height, checking that the height does not go below *n*, and that is left either at *n* or *n*+*k* (depending on the final instruction of the block and the return type of the function) at the end of the block.

## Type Safety

The second phase of the analysis checks that each operation, primitive or defined function, is invoked with arguments of appropriate types.  The operands of an operation are values located either in a local variable or on the stack.  The types of local variables of a function are already provided in the bytecode.  However, the types of stack values are inferred.  This inference and the type checking of each operation can be done separately for each block.  Since the stack height at the beginning of each block is *n* and does not go below *n* during the execution of the block, we only need to model the suffix of the stack starting at *n* for type checking the block instructions.  We model this suffix using a stack of types on which types are pushed and popped as the instruction stream in a block is processed.  The type stack and the statically-known types of local variables are enough to type check each instruction.

## Resource Safety

Resources represent assets of the blockchain. As such, there are certain restrictions over these types that do not apply to normal values. Intuitively, resource values cannot be copied and must be used by the end of the transaction (this means moved to global storage or destroyed). Concretely, the following restrictions apply:

* `CopyLoc` and `StLoc` require the type of local is not of resource kind.
* `WriteRef`, `Eq`, and `Neq` require the type of the reference is not of resource kind.
* At the end of a function (when `Ret` is reached), no any local whose type is of resource kind must be empty, i.e., the value must have been moved out of the local.

As mentioned above, this last rule around `Ret` implies that the resource *must* have been either:

* Moved to global storage via `MoveToSender`.
* Destroyed via `Unpack`.

Both of `MoveToSender` and `Unpack` are internal to the module in which the resource is declared.

## Reference Safety

References are first-class in the bytecode language.  Fresh references become available to a function in several ways:

* Input parameters.
* Taking address of the value in a local variable.
* Taking address of the globally published value in an address.
* Taking address of a field from a reference to the containing struct.
* Return value from a function.

The goal of reference safety checking is to ensure that there are no dangling references.  Here are some examples of dangling references:

* Local variable `y` contains a reference to the value in a local variable `x`; `x` is then moved.
* Local variable `y` contains a reference to the value in a local variable `x`; `x` is then bound to a new value.
* Reference is taken to a local variable that has not been initialized.
* Reference to a value in a local variable is returned from a function.
* Reference `r` is taken to a globally published value `v`; `v` is then unpublished.

References can be either exclusive or shared; the latter allow only read access.  A secondary goal of reference safety checking is to ensure that in the execution context of the bytecode program  — including the entire evaluation stack and all function frames — if there are two distinct storage locations containing references `r1` and `r2` such that `r2` extends `r1`, then both of the following conditions hold:

* If `r1` is tagged exclusive, then it must be inactive, i.e. it is impossible to reach a control location where `r1` is dereferenced or mutated.
* If `r1` is shared, then `r2` is shared.

The two conditions above establish the property of referential transparency, important for scalable program verification, which looks roughly as follows: consider the piece of code `v1 = *r; S; v2 = *r`, where `S` is an arbitrary computation that does not perform any write through the syntactic reference `r` (and no writes to any `r'` that extends `r`).  Then `v1 == v2`.

**Analysis Setup.** The reference safety analysis is set up as a flow analysis (or abstract interpretation for those that are familiar with the concept).  An abstract state is defined for abstractly executing the code of a basic block.  A map is maintained from basic blocks to abstract states.  Given an abstract state *S* at the beginning of a basic block *B*, the abstract execution of *B* results in state *S'*.  This state *S'* is propagated to all successors of *B* and recorded in the map.  If a state already existed for a block, the freshly propagated state is “joined” with the existing state.  The join might fail in which case an error is reported.  If the join succeeds but the abstract state remains unchanged, no further propagation is done.  Otherwise, the state is updated and propagated again through the block.  An error may also be reported when an instruction is processed during propagation of abstract state through a block.  This propagation terminates because ...

**Abstract State.** The abstract state has three components:

* A partial map from locals to abstract values.  Locals not in the domain of this map are unavailable.  Availability is a generalization of the concept of being initialized.  A local variable may become unavailable subsequent to initialization as a result of being moved.  An abstract value is either *Reference*(*n*) (for variables of reference type) or *Value*(*ns*) (for variables of value type), where *n* is a nonce and *ns* is a set of nonces.  A nonce is a constant used to represent a reference.  Let *Nonce* represent the set of all nonces.  If a local variable *l* is mapped to *Value*(*ns*), it means that there are outstanding borrowed references pointing into the value stored in *l*.  For each member *n* of *ns*, there must be a local variable *l* mapped to *Reference*(*n*).  If a local variable *x* is mapped to *Reference*(*n*) and there are local variables *y* and *z* mapped to *Value*(*ns1*) and *Value*(*ns2*) respectively, then it is possible that *n* is a member of both *ns1* and *ns2*.  This simply means that the analysis is lossy.  The special case when *l* is mapped to *Value*({}) means that there are no borrowed references to *l*, and, therefore, *l* may be destroyed or moved.
* The partial map from locals to abstract values is not enough by itself to check bytecode programs because values manipulated by the bytecode can be large nested structures with references pointing into the middle.  A reference pointing into the middle of a value could be extended to get another reference.  Some extensions should be allowed but others should not.  To keep track of relative extensions among references, we have a second component to the abstract state.  This component is a map from nonces to one of two kinds of borrow information: either a set of nonces or a map from fields to sets of nonces.  The current implementation stores this information as two separate maps with disjoint domains:
  1. *borrowed_by* maps from *Nonce* to *Set*<*Nonce*>.
  2. *fields_borrowed_by* maps from *Nonce* to *Map*<*Field*, *Set*<*Nonce*>>.
      * If *n2* in *borrowed_by*[*n1*], then it means that the reference represented by *n2* is an extension of the reference represented by *n1*.
      * If *n2* in *fields_borrowed_by*[*n1*][*f*], it means that the reference represented by *n2* is an extension of the *f*-extension of the reference represented by *n1*.  Based on this intuition, it is a sound overapproximation to move a nonce *n* from the domain of *fields_borrowed_by* to the domain of *borrowed_by* by taking the union of all nonce sets corresponding to all fields in the domain of *fields_borrowed_by*[*n*].
* To propagate an abstract state across the instructions in a block, the values and references on the stack must also be modeled.  We had earlier described how we model the usable stack suffix as a stack of types.  We now augment the contents of this stack to be a structure containing a type and an abstract value.  We maintain the invariant that non-reference values on the stack cannot have pending borrows on them.  Therefore, if there is an abstract value *Value*(*ns*) on the stack, then *ns* is empty.

**Values and References.** Let us take a closer look at how values and references, shared and exclusive, are modeled.

* A non-reference value is modeled as *Value*(*ns*) where *ns* is a set of nonces representing borrowed references.  Destruction/move/copy of this value is deemed safe only if *ns* is empty.  Values on the stack trivially satisfy this property, but values in local variables may not.
* A reference is modeled as *Reference*(*n*), where *n* is a nonce.  If the reference is tagged shared, then read accesses are always allowed and write accesses are never allowed.  If a reference *Reference*(*n*) is tagged exclusive, write access is allowed only if *n* does not have a borrow, and read access is allowed if all nonces that borrow from *n* reside in references that are tagged as shared.  Furthermore, the rules for constructing references guarantee that an extension of a reference tagged shared must also be tagged shared.  Together, these checks provide the property of referential transparency mentioned earlier.

At the moment, the bytecode language does not contain any direct constructors for shared references. `BorrowLoc` and `BorrowGlobal` create exclusive references.  `BorrowField` creates a reference that inherits its tag from the source reference.  Move (when applied to a local containing a reference) moves the reference from a local variable to the stack.  `FreezeRef` is used to convert an existing exclusive reference to a shared reference. In the future, we may add a version of `BorrowGlobal` that generates a shared reference

**Errors.** As mentioned before, an error is reported by the checker in one of the following situations:

* An instruction cannot be proved safe during propagating of abstract state through a block.
* Join of abstract states propagated via different incoming edges into a block fails.

Let us take a closer look at the second reason for error reporting above.  Note that the stack of type and abstract value pairs representing the usable stack suffix is empty at the beginning of a block.  So, the join occurs only over the abstract state representing the available local variables and the borrow information.  The join fails only in the situation when the set of available local variables is different on the two edges.  If the set of available variables is identical, the join itself is straightforward---the borrow sets are unioned point-wise.   There are two subtleties worth mentioning though:

* The set of nonces used in the abstract states along the two edges may not have any connection with each other.  Since the actual nonce values are immaterial, the nonces are canonically mapped to fixed integers (indices of local variables containing the nonces) before performing the join.
* During the join, if a nonce *n* is in the domain of borrowed_by on one side and in the domain of fields_borrowed_by on the other side, *n* is moved from fields_borrowed_by to borrowed_by before doing the join.

**Borrowing References.** Each of the reference constructors ---`BorrowLoc`, `BorrowField`, `BorrowGlobal`, `FreezeRef`, and `CopyLoc`--- is modeled via the generation of a fresh nonce.  While `BorrowLoc` borrows from a value in a local variable, `BorrowGlobal` borrows from the global pool of values.  `BorrowField`, `FreezeRef`, and `CopyLoc` (when applied to a local containing a reference) borrow from the source reference.  Since each fresh nonce is distinct from all previously-generated nonces, the analysis maintains the invariant that all available local variables and stack locations of reference type have distinct nonces representing their abstract value.  Another important invariant is that every nonce referred to in the borrow information must reside in some abstract value representing a local variable or a stack location.

**Releasing References.** References, both global and local, are released by the `ReleaseRef` operation.  References must be explicitly released.  It is an error to return from a function with unreleased references in a local variable of the function.  All references must be explicitly released.  Therefore, it is an error to overwrite an available reference using the `StLoc` operation.

References are implicitly released when consumed by the operations `ReadRef`, `WriteRef`, `Eq`, `Neq`, and `EmitEvent`.

**Global References.** The safety of global references depends on a combination of static and dynamic analysis.  The static analysis does not distinguish between global and local references.  But the dynamic analysis distinguishes between them and performs reference counting on the global references as follows: the bytecode interpreter maintains a map `M` from a pair of Address and fully-qualified resource type to a union (Rust enum) comprising the following values:

* `Empty`
* `RefCount(n)` for some `n` >= 0

Extra state updates and checks are performed by the interpreter for the following operations.  In the code below, assert failure indicates programmer error, and panic failure indicates internal error in interpreter.

```text
MoveFrom<T>(addr) {
    assert M[addr, T] == RefCount(0);
    M[addr, T] := Empty;
}

MoveToSender<T>(addr) {
    assert M[addr, T] == Empty;
    M[addr, T] := RefCount(0);
}

BorrowGlobal<T>(addr) {
    if let RefCount(n) = M[addr, T] then {
        assert n == 0;
        M[addr, T] := RefCount(n+1);
    } else {
        assert false;
    }
}

CopyLoc(ref) {
    if let Global(addr, T) = ref {
        if let RefCount(n) = M[addr, T] then {
            assert n > 0;
            M[addr, T] := RefCount(n+1);
        } else {
            panic false;
        }
    }
}

ReleaseRef(ref) {
    if let Global(addr, T) = ref {
        if let RefCount(n) = M[addr, T] then {
            assert n > 0;
            M[addr, T] := RefCount(n-1);
        } else {
            panic false;
        }
    }
}
```

A subtle point not explicated by the rules above is that `BorrowField` and `FreezeRef`, when applied to a global reference, leave the reference count unchanged.  The reason is because these instructions consume the reference at the top of the stack while producing an extension of it at the top of the stack.  Similarly, since `ReadRef`, `WriteRef`, `Eq`, `Neq`, and `EmitEvent` consume the reference at the top of the stack, they will reduce the reference count by 1.

## Folder Structure

```text
*
├── invalid_mutations  # Library used by proptests
├── src                # Core bytecode verifier files
├── tests              # Proptests
```
