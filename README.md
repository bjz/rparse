## Description
rparse is a parser combinator library written in the [Rust](http://www.rust-lang.org) programming
language. The library consists of parse functions that can be composed together to create arbitrarily 
powerful parsers. The design is such that it is easy for users to define their own parse functions (e.g. 
to define a custom whitespace parser which includes support for a particular comment style).

The parse functions all take a state record as input containing the text to be parsed as well as how much 
has been parsed. They return a result that is either passed or failed. If passed the result includes a new 
state record and a generic T value. If failed the result consists of the input state and an error string.

Most of the built in parse functions take more than just a state argument. Those extra arguments must 
be bound before the parse function can be composed with other functions. For example, 
`bind(repeat_one, _, hex_digit)` creates a parser that consumes one or more hexadecimal digits.

## Example
Here is an example of a simple parser which can be used to evaluate mathematical expressions.

    fn expr_parser() -> str_parser<int>
    {
        let s = space_zero_or_more(_);
    
        // Create closures for parsers which parse a literal followed by optional whitespace.
        let left_paren = literal(_, "(", s);
        let right_paren = literal(_, ")", s);
        let plus_sign = literal(_, "+", s);
        let minus_sign = literal(_, "-", s);
        let mult_sign = literal(_, "*", s);
        let div_sign = literal(_, "/", s);
        let int_literal = integer(_, s, {|v| v});    // fancier parsers can return stuff like int_node
        
        // Parenthesized expressions require a forward reference to the expr parser
        // so we initialize a function pointer to something that always fails, create
        // a parser using the parser expr_ptr points to, and fixup expr_ptr later.
        let expr_ptr = @mut fails(_);
        let expr_ref = forward_ref(_, expr_ptr);    
        
        // sub_expr := '(' expr ')'
        // If sequence is called with [p1, p2] parsers and it succeeds then it will
        // call the closure with [input value, p1 value, p2 value]. In this case the
        // values will be ints (in general they can by anything which is copyable).
        let sub_expr = sequence(_, [left_paren, expr_ref, right_paren], {|values| result::ok(values[1])});
        
        // factor := [-+]? (integer | sub_expr)
        let factor = alternative(_, [
            sequence(_, [plus_sign, sub_expr], {|values| result::ok(values[1])}),
            sequence(_, [minus_sign, sub_expr], {|values| result::ok(-values[1])}),
            int_literal,        // int literal handles leading sign character already
            sub_expr]);
        
        // term := factor ([*/] factor)*
        // Generic arguments are currently always passed by pointer so we need 
        // the lame && sigil.
        let mult = fn@ (&&x: int, &&y: int) -> result<int, str> {ret result::ok(x * y)};
        let div = fn@ (&&x: int, &&y: int) -> result<int, str> {ret if (y != 0) {result::ok(x / y)} else {result::err("divide by zero")}};
        let term = binary_op(_, factor, [
            (mult_sign, factor, mult),
            (div_sign, factor, div)]);
        
        // expr := term ([+-] term)*
        let add = fn@ (&&x: int, &&y: int) -> result<int, str> {ret result::ok(x + y)};
        let sub = fn@ (&&x: int, &&y: int) -> result<int, str> {ret result::ok(x - y)};
        let expr = binary_op(_, term, [
            (plus_sign, term, add),
            (minus_sign, term, sub)]);
        *expr_ptr = expr;
        
        // start := s expr
        // Returns a parser which takes a str and parses leading whitespace followed 
        // by expr. The parser fails if expr does not consume all the input.
        ret everything("unit test", expr, s, 0, _);
    }

Usage looks like this:

    fn expr_parser() -> str_parser<int>
    {
        let parser = expr_parser();
        alt parser("(2 + 3)*4")
        {
            result::ok(value)
            {
                io::stdout().write_line(#fmt["value = %d", value]);
            }
            result::err(error)
            {
                io::stderr().write_line(#fmt["Error '%s' on line %u", error.mesg, error.output.line]);
            }
        }
    }