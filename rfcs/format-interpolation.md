# Format interpolation

This RFC was designed with @Octachron. We propose a new feature of
format strings, as used by the `Printf` and `Format` module, to
support interpolation of format strings. This interacts with OCaml
lexing rules, so it needs language support.

In short, the idea is to support `%{d: foo %}` in format strings,
where `foo` must be an integer as `d` is an integer format specifier
(`%d` takes an integer). The specifier `%a` takes two arguments
(say `pp` and `foo`), so the interpolation syntax would use a `%:`
separator: `%{a: pp %: foo %}`.


### Simple example

Before (from `asmcomp/cmm.ml`):

```ocaml
Misc.fatal_errorf "Cannot set label counter to %d, it must be >= %d"
  l !label_counter ()
```

After:
```ocaml
Misc.fatal_errorf "Cannot set label counter to %{d: l %}, \
                   it must be >= %{d: !label_counter %}"
```


### More complex example

Before (from `parsing/pprintast.ml`):

```ocaml
| Pcf_inherit (ovf, ce, so) ->
    pp f "@[<2>inherit@ %s@ %a%a@]%a" (override ovf)
      (class_expr ctxt) ce
      (fun f so -> match so with
         | None -> ();
         | Some (s) -> pp f "@ as %a" ident_of_name s.txt ) so
      (item_attributes ctxt) x.pcf_attributes
```

After:

```ocaml
| Pcf_inherit (ovf, ce, so) ->
    let rename f = function
      | None -> ()
      | Some s -> pp f "@ as %a" ident_of_name s.txt
    in
    pp f "@[<2>inherit@ %{s: override ovf %}@ \
            %{a: class_expr ctx %: ce %}\
            %{a: rename %: so %}\
          @]%{a: item_attributes ctx %: x.pcf_attributes %}"
```

### Why?

We believe that the interpolation syntax is easier to read, write and maintain in many situations. For larger format strings in particular (which are common when using Format, as we write format strings will well-nested pretty-printing boxex), it can be difficult to find which argument are related to which format. For example, if you decide to insert a new format specifier in the middle of the format string, it is not obvious where to place the corresponding arguments.

## Related work

TODO check:
- other languages: Rust, Scala, Python, Go
- camlp4 and ppx extensions for format strings, in particular ppx_format.

### Interpolation in OCaml

### Other languages with interpolation support

## Implementation ideas

One potential blocker for string interpolation is our ability to implement it. I discussed a few implementation approaches with @Octachron. One approach that we think would be workable is to extend the lexer with tokens to describe string literal fragments in-between interpolation points.

Currently the string literal `"foo"` maps to a single lexer token `STRING "\"foo\""`. For the interpolated `"Hello %{ ... %}!\n"`, we would use first a `OPEN_STRING_OPEN_INTERP "\"Hello %{"` token, then follow with the normal OCaml lexemes of the `...` part, then finally a `CLOSE_INTERP_CLOSE_STRING "%}!\n\""` token. There would also be a token `CLOSE_INTERP_OPEN_INTERP "%} blah %{"` when strings have several interpolation points.

The parser can easily parse this lexeme string into a more structured construct, such as

```ocaml
| Pexp_interpolated_string of (string_literal * interpolated_expression) list * string_literal
```

This construction is *not* valid at the type `string`, it can only be used to build `_ format6` GADT values, and it is interpreted then by a new GADT constructor for interpolated fragments within the format string.
