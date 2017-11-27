---
layout: post
title:  "Build a Rule Validation Engine with Go-Yacc"
date:   2017-10-15 08:00:00 +0800
categories: go compiler
---

Problem Description:
====================

Suppose we have a bunch of items labeled with description illustrating under which conditions that item is valid. 

The input of this problem includes:

* a set of items labled with description mentioned above(**here we call them rules**)
* and a set of data describing a certain **condition**.

The output of this problems:

* a set of items that are valid under that condition

There is a real example in one of my projects.

Items are all labeld with the following structural description:

{% highlight javascript %}
    //The data below means that item is valid for software that is between version 1.5.0 and 2.0.0, 
    //the user for that software has a car, 
    //as well as the user is located in a city that has a code included by the list.
    {
        "version": {
            "max": "2.0.0",
            "min": "1.5.0"
        },
        "driver_tag": "yes",
        "city_code": {
            "compare": "included",
            "value": [238,593,285]
        }
    }
    //And suppose we can initially have user information like this:
    {
        "version": "1.6.1",
        "driver_tag": "no",
        "city_code": 593
    }
    //Obviously that item is valid for this user.
{% endhighlight %}

__Notice: The entries both for describing an item and a user can be numerous and extensive. The total number of description tags increased as the software system upgrades__

A simple version of a validation algorithm in Go language would look like this:

{% highlight go %}
    func validate(user *UserInfomation, rules *ItemRules) bool {
        if rules.Version != nil {
            //if there is a definition of versionValidate function
            if ok := versionValidate(user.Version, rules.Version.Min, rules.Version.Max); !ok {
                return false
            }
        }
        if rules.DriverTag != "" {
            if user.DriverTag != rules.DriverTag {
                return false
            }
        }
        if rules.CityCode != nil {
            //if there is a definition of cityCodeValidate function
            if ok := cityCodeValidate(user.CityCode, rules.CityCode); !ok {
                return false
            } 
        }

        return true
    }
{% endhighlight %}

We can image that the validate function is consist of numerous similar snippets of code(only three snippets are shown here!). What troubles me here is the size of the function makes it no easy for reading and duplicate is inevitabe if I have to do a similar validation(eg. validation for other users' tags). 

Therefore I want to build a machine to automatically validate the items. Here I used go-yacc to achieve this goal.

1.Redefine items' rule
=======================

If the rules are in the form of __"expression"__, then there are three basic operations when dealing with rule:

* Equality or inequality checking: like `someTag == "tag1"` or `someTag != "tag2"`
* Range checking: like `minVersion <= version <= maxVersion`
* Enumeration checking: like `inEnumberation(cityCode, cityCodeList)`
* Finally the rules can be combined with logical `AND` and `OR`

Next I defined syntax for describing these expression and it turned out to be very analogous to mathematical expression for the basic operations are similar to mathematical operation; there is precedence when calculating different components of an expression.

The syntax for mathematical expression included only + , -, * and / is look like this in Context-Free-Language:

`E -> E + T | E - T | T`

`T -> T * F | T / F | F`

`F -> (E) | id`

id is the terminal for mathematical expression but here for this case id here is a simple rule corresponding to an rule entry.

`E -> E || E1 | E1`

`E1 -> E1 && E2 | E2`

`E2 -> (E) | R` // R for rule

Here is an extension of syntax for mathematical expression

`R -> EQUALITY | ENUMERATION | RANGE`

`EQUALITY -> tag == INDETIFER | tag != INDETIFER`

`ENUMERATION -> city IN [CITY_LIST] | city != [CITY_LIST]`

`RANGE -> version IN [VERSION, VERSION] | version NIN [VERSION, VERSION]`

`CITY_LIST -> CITY_LIST , CITY_LIST_ITEM | CITY_LIST_ITEM`

`CITY_LIST_ITEM -> NUMBER`

`VERSION -> NUMBER . NUMBER . NUMBER`

`NUMBER -> NUMBER DIGIT | DIGIT`

`INDETIFER -> INDETIFER letter | letter`


Here __tag,city,version,letter,==,!=,[,],IN,NIN__ are all terminals.

2.Build a Parser
=======================
There are three parts of a formal yacc source file. It is the same as in go-yacc

create a go-yacc source file rule.y

The first part consists of some predefinement here. As for Go, it is the package declaration and imports.

{% highlight go %}
    %{
        package main
        import (
            "unicode"
        )
    }%
{% endhighlight %}

The second part mainly defines the parser with types of terminals and nonterminals.

{% highlight go %}
    //the union defines all the possible data type for terminals and nonterminals.
    //here we have boolean, integer, slice of integers and string
    //define nicknames for data types
    %union {
        rst bool
        num int
        num_list []int
        str string
    }

    //Type definition for Nonterminals
    %type <rst> E E1 E2 R EQUALITY RANGE ENUMERATION // they are all in type boolean
    %type <num> CITY_LIST_ITEM
    %type <num_list> CITY_LIST
    ....

    //Typ definition for Terminals
    %token <str> tag
    %token <num> DIGIT
    ...
    
    %start E

    %%
{% endhighlight %}

The third part defines the derivations(already shown above) and attaches action for every derivation(It helps to calculate the value of a nonterminal).

{% highlight go %}

E       : E OR E1
        {
            $$ = $1 || $3
            //E and E1 are all boolean
        }
    |   E1
        {
            $$ = $1
        }
    ;
E1      : E1 AND E2
        {
            $$ = $1 && $3
        }
    |   E2
        {
            $$ = $1
        }
    ;
...

Rule    : Range
            {
                $$ = $1
            }
        |   Enumeration
            {
                $$ = $1
            }
        |   Equality
            {
                $$ = $1
            }
        ;

ENUMERATION : CITY IN CITY_LIST 
            {
                $$ = false
                for _, n := range $3 {
                    if n == $1 {
                        $$ = true
                        break
                    }
                } 
            }
            | CITY NIN CITY_LIST
            {
                $$ = true

                for _, n := range $3 {
                    if n == $1 {
                        $$ = false
                        break
                    }
                } 
            }
        ;

...

{% endhighlight %}

The tool needs a self-custom lexical analyzer. However, it is difficult to write a correct token recognition function. Lucky enough, the language to describe the tokens here can be covered by context-free language too. So the derivations are supplemented by token derivations like this `NUMBER -> NUMBER | DIGIT` so that I don't have to retrieve a real number from the text. All I have to do is to retrieve a single character and recognize whether it is a letter or a digit or a prefix of a certain terminal like '&&'.

{% highlight go %}

var lexKeywords = map[string]int{
    "&&": AND,
    "||" : OR,
	"IN":  IN,
	"NIN": NIN,
	"==":  EQ,
	"!=":  NEQ,
    "(" : LP,
    ")" : RP,
    "[" : LB,
    "]" : RB,
    "," : COMMA,
    "tag" : TAG,
    "city" : CITY,
}

var prefixDictionary = map[string]string {
    "&": true,
    "|": true,
    //all of the prefix of the terminals
}

//It's our main lexer
type RuleLex struct {
	pos         int    // lexical position used when doing lexical analysis
	s           string //lexical string
    Data        *ContextData  // context data used to replace context constant in the derivations
	Rst         bool   // rule match result
	SyntaxError error  // syntax error
}

func (l *RuleLex) Lex(lval *ContextSymType) int {

	var c rune = ' '

    //skip space
	for c == ' ' {
		if l.pos == len(l.s) {
			return 0
		}
		c = rune(l.s[l.pos])
		l.pos += 1
	}

	if unicode.IsDigit(c) {

		lval.num = int(c) - '0'
		return DIGIT

	} else {

		str := string(c)

        if _, isPrefix := prefixDictionary[str]; isPrefix {
            count := 0
            for l.pos < len(l.s) { 
                if l.s[l.pos] == ' ' {
                    break
                }
                c = rune(l.s[l.pos])
                str += string(c)
                l.pos += 1 
                count += 1

                //pre-defined tokens
                if keyword, isKeyword := lexKeywords[str]; isKeyword {
                    lval.str = str 
                    return keyword
                }
            }
            
            l.pos -= count
        }

        //otherwise identifier
        lval.str = string(c)
        return IDENTIFIER
	} 
}

//Define a error handler
func (l *RuleLex) Error(s string) {
	l.SyntaxError = errors.New(s)
}

func (l *RuleLex) GetResult() bool {
    return l.rst
}

{% endhighlight %}

3.Compile the Parser to output Parser Source Code
=================================================

{% highlight shell %}

go tool yacc -o rule.go -p Rule rule.y

{% endhighlight %}

[go-yacc command and parameters](https://godoc.org/github.com/cznic/goyacc "With a Title").

Notice: the -p option defines the Prefix of the Parser Object so that type RuleLex is automatically generated in the output file rule.go and used in the parsing procedure.


Run the parser like this:

{% highlight go %}

    package main

    import (
        "context/compiler"
        "fmt"
    )

    func main() {

        test := "C IN [100,101,110]"

        ruleLex := new(compiler.RuleLex)

        data := new(compiler.RuleData)
        data.CityCode = 110
        data.Version = "1.0.0"
        data.Tag = "tt"

        ruleLex.Init(test, data)
        compiler.RuleParse(ruleLex) // rule the parser
        rst := ruleLex.GetResult()

        fmt.Println("Context " + test + " is matching...")

        if contextLex.SyntaxError != nil {
            fmt.Println("Error is detected:", contextLex.SyntaxError)
        } else {
            fmt.Println("Rule match result:", contextLex.Rst)
        }
    }

{% endhighlight %}
