<h1> <img src="https://craftypixels.com/placeholder-image/60x20/2f6faf/2f6faf"> PostScript </h1>
This is an implementation of PostScript language level 3. The full language is implemented except for graphics operators.

Most people may know [PostScript](https://en.wikipedia.org/wiki/PostScript) as a page description language for printers. But actually PostScript is a full fledged, turing complete programming language with dedicated support for vector graphics and fonts.

This implementation is about the programming language without the graphical aspects. PostScript is a stack based language, a descendent of [Forth](https://en.wikipedia.org/wiki/Forth_(programming_language)) and resembling the famous [HP](https://en.wikipedia.org/wiki/HP_calculators) [RPN](https://en.wikipedia.org/wiki/Reverse_Polish_notation) (Reverse_Polish_notation) calculator programming model.

## Get the code
Load the packages **[Values]** and **[PostScript]** from the [Cincom Public Store](https://github.com/PDFtalk/.github/wiki/Getting-Started#5-set-up-access-to-the-public-store) into your [VisualWorks](http://www.cincomsmalltalk.com/main/products/visualworks/) image. Tests are in package **[PostScript Testing]**. The package can be used stand-alone without PDFtalk. It only depends on the **[Values]** package.

The package is now part of the **{PDFtalk}** bundle.

## Use it

```
| ps |
ps := PostScript.Interpreter run: '3 4 add'.
```

returns an instance of the interpreter which consumed and processed the PostScript language string.

```
ps pop
```

returns and removes the top of the operand stack - in this case **7**.

You can also run successive pieces of PostScript code:

```
| ps |
ps := PostScript.Interpreter run: '3 4 add'.
ps run: 'dup mul'.
ps stack top. "is 49"
```

## Motivation
PDF is the successor of PostScript and still depends on it in some places:

* CMaps in fonts
* Type1 fonts
* PostScript calculator functions
* PostScript XObjects (deprecated)

Ultimately I want to extract text from PDFs for which I need CMaps.

Also, I like PostScript. I used it quite a bit to write UIs with Display PostScript on a Sun [NeWS](https://en.wikipedia.org/wiki/NeWS) workstation. The language allows for some nice programming patterns.

## PostScript programming introduction
To add two numbers, the numbers are entered first and then `add`:

```
3 4 add
```

This works with a stack, the so called `operand stack` (or just stack):

```
        % ||  <       Initial empty operand stack
3       % || 3 <      the first number is on the stack
4       % || 3 4 <    the second number is pushed onto the stack
add     % || 7 <      the ''add'' operator takes 2 numbers and pushes the sum onto the stack
```

In the Smalltalk implementation you do this with:

```
| ps |
ps := PostScript.Interpreter run: '3 4 add'.   "this returns an Interpreter"
ps pop                                         "returns the top element of the stack: 7"
```

There are no variables in PostScript, not even temporary ones. Instead, objects can be stores in a dictionary on the `dictionary stack`:

```
/seven { 3 4 add } def
```

This stores the procedure `{3 4 add}` under the name `/seven` in the top dictionary of the dictionary stack. The new operator can be used to put a 7 onto the stack:

```
seven dup mul
```

leaves 49 on the stack.

At the start of a PostScript interpreter, the dictionary stack contains 3 dictionaries:

* **user dict**: “empty dictionary for user defined entries”
* **global dict**: “globally accessible definitions. Initially empty”
* **system dict**: “bottom of the stack. Read only with build-in operators”

New definitions are put into the topmost dictionary of the stack. This allows programs to override system definitions. The NeWS system from Sun used this to implement an object oriented Display PostScript with the dictionaries as classes. Another view is to see the dictionaries as namespaces.

The dictionaries can be named with an interesting trick: the dictionary itself is stored under a key which is interpreted as its name. The system dict for example is constructed like:

```
| dict |
dict := PSDictionary new.
dict at: #systemdict put: dict.
^dict
```

This creates a recursive structure which is not handled well by the standard Smalltalk dictionary, hence `PSDictionary` as my implementation of it.

```
Interestingly, Gemstone has exactly the same dictionaries implementing method lookup.
Including the naming scheme for dictionaries!
```

(more will be written on demand…)

## Implementation notes
This implementation does not have any graphics operators; it is just about the programming language.

There is a distinction about local and global VM, which, in my view, is an optimization not important for the language. Therefore, I did not implement this distinction (and hope that it really does not have any importance…).

## Exception handling example
When looking at CMaps in the wild, I encountered one where the PostScript was wrong. Instead of

```
/CIDInit /ProcSet findresource begin 12 dict begin begincmap    % ...
```

the program started with

```
/CIDInit /ProcSet find begin 12 dict begin begincmap    % ...
```

Instead of `findresource` the non existing operator `find` was used.

To fix this I tried two approaches: the PostScript and the Smalltalk way.

In PostScript, you can just define the missing `find` operator and read it again:

```
| source ps resources |
source := self exampleWrongSource.
ps := Interpreter new.
resources := [(ps run: source) resources] on: KeyNotFoundError do: [:ex |
	(ex selector = #valueAt: and: [
	ex parameter = #find])
			ifTrue: [
				| ps1 |
				ps1 := Interpreter run: '/find /findresource cvx def'.
				ex return: (ps1 run: source) resources]
			ifFalse: [
				ex pass]].
^self newWith: ((resources at: #CMap) at: #F1)
```

In Smalltalk you could just call the correct operator and resume:

```
| source ps resources |
source := self exampleWrongSource.
ps := Interpreter new.
resources := [(ps run: source) resources] on: KeyNotFoundError do: [:ex |
	(ex selector = #valueAt: and: [
	ex parameter = #find])
		ifTrue: [ex resume: (ex receiver valueAt: #findresource)]
		ifFalse: [ex pass]].
^self newWith: ((resources at: #CMap) at: #F1)
```

I think that both approaches are very elegant.
