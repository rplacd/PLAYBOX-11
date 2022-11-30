PLAYBOX-11 handbook
===================

PLAYBOX-11 is a procedural, statement-based, interactive, dynamically-bound,
native code compiled, resource-light systems-language programming environment
for the PDP-11.

It is not as expressive as C (or even B/BCPL). Instead, it "reskins" and adds
very limited syntactic sugar to MACRO-11: *functions*, *subroutines*, *let statements*.
Because PLAYBOX-11 is late-bound, all functions, subroutines, etc. are found in
a lookup table.

PLAYBOX-11 has a monitor where PLAYBOX-11 input can be edited and compiled,
and where PLAYBOX-11 statements can be run. 

Each statement compiles to a *fixed number* of MACRO-11 instructions. In particular,
arbitrary expression computation is not allowed.

It is designed to implement systems programs – virtual machines, etc. You will be able
to incrementally edit, compile, and run PLAYBOX-11 code at a PDP-11 terminal.
The core language and environment does not have access to a heap; only a
terminal.

Exposition of language features + philosophy: examples
------------------------------------------------------

Here are examples of PLAYBOX-11 code, that demonstrate the philosophy
of the language: limited syntactic sugar for MACRO-11.
Once we have a text input facility in the PLAYBOX-11 monitor,
this whole transcript below can be entered, verbatim, into the PLAYBOX-11 monitor.

Feel free to experiment with your own! The syntax below will help.

```
; A fibonacci function, that demonstrates
; usage of the stack, and statically allocated
; variables (which are the only type)
(def/fn fibonacci (times)	    ; Function arguments are passed and returned via
                                ; a global stack, with a well-defined calling
                                ; convention.
                                ;
                                ; Like MACRO-11, we are conservative about types.
                                ; In fact, we only support words.

    (let/lit ((n #0) (m #1)); Variables can only be declared and initialized in 'let' forms –
                                ;  `let-lit`, `let-jsr`. Each one allows a very specific
                                ; form of initialization - literal, function call. There is no
                                ; `let` form that allows arbitrary computations.
                                ;
                                ; Additionally, we inherit conventions from MACRO-11,
                                ; like octal immediates by default (since we want
                                ; to be able to write compiled languages +
                                ; easily output PDP-11 machine code).

                            
        
        (cmp\until-after (eq times #1) ; (see below)
                                ; PLAYBOX-11 has structured control flow.
                                ; This will compile to the MACRO-11:
                                ;
                                ; UNTIL: CMP TIMES, #0
                                ; 		 BEQ AFTER
                                ; 		 ...
                                ; 		 BR UNTIL
                                ; AFTER: ...
                                ;
                                ; Also, note the use of '\' instead of '/'. In general,
                                ; statements in PLAYBOX-11 that hinge around an existing
                                ; MACRO-11 statement use '\'s; while statements (like
                                ; scoped declarations) that have no equivalent in
                                ; MACRO-11 use '/'s.

            (mov n m)			; Most if not all primitive functions in PLAYBOX-11
            (inc m)				; correspond to single instructions.

            )					; At this point, we return back to the `until`.

        (rts\with m)			; Functions return values on the stack.
    )
)


; A simple demonstration of function calls and our two conditions:
;  `cmp\until-after` and `cmp\if`:
; a test for divisibility, both by 2 and by 3, where the two
; divisibility tests are implemented in separate calls.
(def/fn divisible-by-2-and-3? (n)
    (let/jsr ((by-2? (divisible-by-2? n))	; Function calling is highly restricted:
              (by-3? (divisible-by-3? n)))	; you can only call a function in the context
                                                ; of `let/jsr` - i.e. when initializing a variable.
                                            ;
                                            ; Also remember (cf. `let/lit` above) that `let/lit`
                                            ; only initializes values with a direct function
                                            ; call. No `let` form initializes with an arbitrary
                                            ; expression.

        (mul by-2? by-3?)					; (return AND of by-2 and by-3.)
        (rts\with by-3?)					;   "  (ditto)
)
(def/fn divisible-by-2? (n)
    (cmp\until-after (ble n 0)
        (sub #2 n)
    )
    (cmp\if (eq n 0)						; `cmp\if` is our second conntrol flow expression.
        ((rts\with 1))						; This will compile to the MACRO-11:
        ((rts\with 0))						; 
    )										; 		CMP N, #0
                                            ; 		BEQ IFTRU
                                            ; 		BR  IFFLS
                                            ; IFTRU: ...
                                            ; IFFLS: ...
                                            ;
                                            ; An NB for users of Lisp: note that `cmp\if` is NOT
                                            ; `if`, in the sense that the two branches of the `if`
                                            ; are not expressions, but consecutive statements wrapped
                                            ; in ()s. The ()s are OBLIGATORY.
                                            ; The syntax of `cmp\if` is, informally,
                                            ; 	(cmp\if (COMPARISON STATEMENT)
                                            ;		((STATEMENT TRUE-1) ...)
                                            ;		((STATEMENT FALSE-1) ...))

)
(def/fn divisible-by-3? (n)
    (cmp\until-after (ble n 0)
        (sub #3 n)
    )
    (cmp\if (eq n 0)
        ((rts\with 1))
        ((rts\with 0))
    )
)

; A more complicated example of function calls: recursive factorial.


; To demonstrate PLAYBOX-11's inheritance from MACRO-11, every addressing
; mode in MACRO-11 is available for use in PLAYBOX-11 operations and initializations.
```

Syntax
-------


Phase 1: an offline PLAYBOX-11 to MACRO-11 transpiler
-----------------------------------------------------