Hey there, it looks like you're trying to parse text input in Objective-C. You've come to the right place.

**ParseKit is a parser generator implemented in Objective-C**, which converts grammars into parsers intended for use in Cocoa applications running on iOS or Mac OS X.

With ParseKit, you can define your language with a **high-level**, **easy-to-use**, **BNF-style grammar**, and ParseKit will **generate a parser** for your langague implemented in **Objective-C source code**.

Specifically, parsers produced by ParseKit are:

* **Recursive descent**
* **Deterministic**
* **[Packrat](http://bford.info/packrat/ "Packrat Parsing and
	Parsing Expression Grammars")** (or *memoizing*), 
* **Infinite-lookahead** 
* Modern **Objective-C** (using blocks, ARC, properties)

That's a mouthful, but what it means in practice is that ParseKit offers you a great deal of flexibility and expressive power when designing your grammars, but also produces parsers which exhibit good (linear) performance characteristics at runtime. Also, the Objective-C code produced by ParseKit is clean, readable, and easy to debug or tweak by hand.

The design of ParseKit has been heavily influenced by [ANTLR](http://antlr.org) and a [book by Stephen J Metsker](http://www.amazon.com/Building-Parsers-Java-Steven-Metsker/dp/0201719622).

In this tutorial, I'll demonstrate how to use ParseKit to implement a small *"MiniMath"* expression language in an iOS application. When we're done, we'll be able to parse *MiniMath* expressions and compute and display the results.

### Designing the Grammar

First, let's define for our *"MiniMath"* language. *MiniMath* should allow expressions like:

    1            // bare numbers
    2 + 2 + 42   // addition (including repetition)
    2 * (2 + 4)  // multiplication and sub-expressions
	(2+2)*3      // should be tolerant of whitespace presence or absence
	3.14 *5      // support for optional floating point numbers

OK, now that we know what the expected *MiniMath* input looks like, let's design a ParseKit grammar to match it. Every ParseKit grammar has to start with a rule called `@start`. Since *MiniMath* is an expression language, let's define our `@start` rule as an expression.

    @start = expr;

But how do we define `expr`?

    expr =  ???  // TODO

Rather than designing our grammar from the top down, let's hold that thought, and work from the bottom up instead.

Working from the bottom, we'll start with a rule called `atom`. And since *MiniMath* deals with numbers, we'll define `atom` as a `Number`.

    atom = Number;

Notice how the rules we define ourselves (like `expr` and `atom`) start with lowercase letters. There are also built-in terminal rules like `Number`, `Word`, `QuotedString` and more which match common token types like numbers, words, and quoted strings. The built-in rules always start with uppercase letters, while the rules we define ourselves must start with lowercase letters.

The built-in `Number` rule matches a series of digits as you would expect. By default, it also matches optional floating-point and exponential parts of a number (this behavior is easily configurable).

Now that we have defined an `atom` rule, let's define a primary expression.

    primary = atom | '(' expr ')';

A `primary` expression is either an atom or a parenthesized sub expression. The parentheses here can be used to alter operator precedence.

Note that we can recursively call our own `expr` rule (although in ParseKit grammars, you must always avoid [left recursion](http://en.wikipedia.org/wiki/Left_recursion)). 

Now let's move on to multiplication and addition. As usual, we want multiplication to bind more tightly than addition. Since we are working from the bottom up, we can make multiplication bind more tightly by defining it first.

Let's define multiplication as a primary expression times a primary expression.

    multExpr = primary '*' primary;

But we want to allow repetition in our multiplication expressions, like `2 * 8 * 0`, so we'll alter our `multExpr` rule slightly by wrapping the operator and the right-hand side operand in an optional repetition using `*`.

    multExpr = primary ('*' primary)*;

Our addition rule will look very similar:

    addExpr = multExpr ('+' multExpr)*;

Since our addition rule is defined in terms of multiplication operands, this will force multiplication to bind more tightly than addition. 

Now we can define our `expr` rule as an addition expression:

    @start = expr;
    expr = addExpr;

Finally, let's update our grammar to discard unnecessary tokens. The post-fix `!` operator can be used to discard a token which is not needed to compute a result. In the case of *MiniMath*, we'll want to discard any token that is not a number (all of the literal strings in our grammar).

Here's the complete grammar:

    @start = expr;
    expr = addExpr;
    addExpr = multExpr ('+'! multExpr)*;
    multExpr = primary ('*'! primary)*;
    primary = atom | '('! expr ')'!;
    atom = Number;
    
### Adding Actions to the Grammar

OK, so we designed a grammar for our *MiniMath* language that can be fed to ParseKit to produce Objective-C source code for our parser.
 
But we don't just want to parse input, we also want to compute a result. The easiest way to do this is to use **grammar actions**. Grammar actions are small pieces of Objective-C source code embedded directly in a ParseKit grammar.

We'll start by adding an Action to the `atom` rule:
 
    atom = Number 
    {
        PKToken *tok = [self.assembly pop]; // pop the Number token
        NSAssert(tok.isNumber, @"a number token just matched in `atom`");
        
        NSNumber *n = @(tok.floatValue);
        [self.assembly push:n];  // push an NSNumber object
    };
    
As you can see, actions are blocks of Objective-C code enclosed in curly braces and placed after any rule reference. 

In any action, there is an `self.assembly` object (of type `PKAssembly`) available which serves as a **stack** (via the `-push:` and `-pop` instance methods). The `self.assembly`'s stack contains the most recently parsed tokens (instances of `PKToken`), and also serves as a place to store your work as you compute the result.

Actions are executed immediately after their preceeding rule matches. So tokens which have recently been matched are available at the top of the assembly's stack.

In this case, we are popping a just-matched number token off the stack, converting it to a float value, and pushing an `NSNumber` back onto the stack for later use.

But our action code is a bit verbose, and is making our grammar harder to read and understand. No problem: ParseKit includes some handy macros that can make this code more concise. Here's the `atom` rule and action rewritten using those macros:

    atom = Number { 
        // pop a token off the stack and push it back as a float value 
        PUSH_FLOAT(POP_FLOAT()); 
    };

This shortened action is exactly equivalent to the more verbose version above. The action still pops a number token off the stack, converts it to a float value, and pushes an `NSNumber` back onto the stack

Now let's add an action to perform multiplication in the `multExpr` rule:

    multExpr = primary ('*'! primary { 
        NSNumber *rhs = [self.assembly pop];
        NSNumber *lhs = [self.assembly pop];
        NSNumber *n = @([lhs floatValue] * [rhs floatValue]);
        [self.assembly push:n];
    })*;

This action executes immediately after the multiply operator (`*`) and right-hand side `primary` operand have been matched. Since the `*` operator has been discarded,  we can be assured that the top 2 objects on the stack are NSNumbers placed by our `atom` rule action.  

Again, we can use ParseKit's handy built-in macros to simplify our Objective-C action code. Here's the same action simplified:

    multExpr = primary ('*'! primary { 
        PUSH_FLOAT(POP_FLOAT() * POP_FLOAT());
    })*;

Finally, we'll need a similar action for our addition expression rule. Here's the complete grammar including actions:

    @start = expr;
    expr = addExpr;
    addExpr = multExpr ('+'! multExpr {
        PUSH_FLOAT(POP_FLOAT() + POP_FLOAT());
    })*;
    multExpr = primary ('*'! primary { 
        PUSH_FLOAT(POP_FLOAT() * POP_FLOAT());
    })*;
    primary = atom | '('! expr ')'!;
    atom = Number { 
        PUSH_FLOAT(POP_FLOAT()); 
    };

### Interlude: Checkout the Example Project (with ParseKit Dependency)

OK, time to [checkout the ParseKit MiniMath Example](https://github.com/itod/ParseKitMiniMathExample/zipball/master) project. This project includes [ParseKit](https://github.com/itod/parsekit) as an external dependency.

### Generating Parser Source Code

Now that our *MiniMath* grammar is complete, we can use ParseKit to generate Objective-C source code for our parser.

Open the **MiniMath** Xcode project, then select and run the **ParserGenApp** target.

**ParserGenApp** is actually a target in the embedded ParseKit sub-project, and is the way you convert your ParseKit grammars into Objective-C source code.

Paste the *MiniMath* grammar into the large text area at the bottom of the ParserGenApp window, and select the options shown below.

![ParserGenApp](http://parsekit.com/github/parsergen.png)

Click the **Generate** button and notice that a [MiniMathParser.h](https://github.com/itod/ParseKitMiniMathExample/blob/master/MiniMath/MiniMathParser.h) [MiniMathParser.m](https://github.com/itod/ParseKitMiniMathExample/blob/master/MiniMath/MiniMathParser.m) file have been created, and appear on your Desktop. Normally, you'd need to drag these source code files into your app's Xcode project, but in the case of *MiniMath*, I've included the files already (cooking show style!).

![Produced Files](http://parsekit.com/github/files.png)

### Run the MiniMath Example iOS App

Back in Xcode, switch to the **MiniMath** target. This target is an example iOS app with an **Input** textfield, **Calc** button and a **Result** textfield:

![MiniMathApp](http://parsekit.com/github/app_empty.png)

Here's the implementation of the `calc:` Action attached to the **Calc** button, showing how to use the `MiniMathParser` we just created:

	- (IBAction)calc:(id)sender {
	    NSString *input = [_inputField text];
    
	    MiniMathParser *parser = [[MiniMathParser alloc] init];
    
	    NSError *err = nil;
	    PKAssembly *result = [parser parseString:input 
		 							   assembler:nil 
									       error:&err];

	    if (!result) {
	        if (err) NSLog(@"%@", err);
	        return;
	    }
    
	    // print the entire assembly in the result output field
	    [_outputField setText:[result description]];
	}

Run the app (make sure you've selected the **iPhone Simulator** as your run destination), and you'll see the input field is pre-populated with an example expression. Click the **Calc** button to compute and display the result:

![MiniMathApp](http://parsekit.com/github/app.png)

This displayed result deserves a bit of explanation. 

The result of the `-[MiniMathParser parseString:assembler:error:]` method is an assembly object of type `PKAssembly` described earlier. Again, an **assembly** is intended to be a convenient place to examine recently-matched tokens as well as store temporary work as the parse executes.

A `PKAssembly` object combines a **stack** (which we've used earlier in this tutorial) and a buffer of the tokens matched in the input string so far. Printing an assembly via the `-[PKAssembly description]` method returns a string with the following format:

  **[** `stack`, `contents`, `here` **]** `matched` **/** `tokens` **/** `here` **^**
  
The contents of the assembly's stack are on the left between the `[` `]` square brackets. And the buffered tokenized input is displayed on the right between `/` slash chars. Each slash separates individual tokens. The `^` caret represents the parser's current cursor position in the input token stream.

So for our result:

    [12](/2/+/2/)/*/3^

`12` is on the stack. And tokens `(`, `2`, `+`, `2`, `)`, `*`, and `3` have been matched. The cursor (`^`) is positioned at the end of the input string (our parse successfully matched the entire string).

This assembly display can often be useful when debugging parsers. But for now, all we want is the numerical result of parsing our *MiniMath* expression. As you can see, the result (`12`), is on the top of the stack. So we can just pop the numerical result off the stack and use it:

    PKAssembly *result = [parser parseString:input assembler:nil error:nil];
	NSNumber *n = [result pop];
    NSLog(@"The numerical result is: %@", n);

For our given input of `(2+2)*3`:

	The numerical result is: 12	

### Conclusion

I hope this simple tutorial has sparked some ideas in your mind for how to use ParseKit for parsing more interesting langauges than *MiniMath* in your Mac and iOS applications.

The [main ParseKit repository is here](http://github.com/itod/parsekit/). I'm [@iTod](https://twitter.com/iTod "Todd Ditchendorf (iTod) on Twitter") on Twitter, and if you find some use for ParseKit, consider checking out [some of my other software](http://celestialteapot.com). Cheers!