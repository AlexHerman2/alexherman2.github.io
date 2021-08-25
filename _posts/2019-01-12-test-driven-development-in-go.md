---
title: "Test Driven Development with Go"
layout: post
---
# Test driven development with Go

### Background
What I’m working on for this project is an API sand boxing tool that can read a Swagger spec and then serve mock responses. API Blueprint has something similar (called [Drakov](https://github.com/Aconex/drakov)) that is super useful, but I haven’t been able to find something comparable for Swagger that doesn’t require an intermediate step of code generation. Besides, this could be kind of a fun thing to make.

This post will focus on just a small piece of that larger (potential) project. What we’re going to do is write a method that takes a Swagger formatted path string (with the curly bracketed parameters) and returns a regex that can determine if a specific request path matches it.

### Getting started
In a previous post, we set up a Go Lang dev environment (with vim-go, delve, and whatnot). We’ll be using that environment and adding Go tests as well. Let’s start by getting go tests installed
```
go get -u github.com/cweill/gotests/...
```
 And the Vim add on to use it:
```
git clone https://github.com/buoto/gotests-vim.git ~/.vim/bundle/gotests-vim
```
### The first test
So, I know that we’re supposed to write a test first,  it we’re going to fudge a bit and start with an interface definition of the path substituter. It’s just going to be one method, but it having an interface instead of just a struct will make testing whatever uses this substituter much easier to test.
```
package path

type URLSubstitutor interface {
    ToRegex(swaggerPath string) string
}
```
Now, let’s make an empty struct that implements that interface. Simple enough

<pre>
<code>
package path

type URLSubstitutor interface {
    ToRegex(swaggerPath string) string
}

<b>type urlSubstitutor struct{}</b>

<b>func (subs urlSubstitutor) ToRegex(swaggerPath string) string {

}</b>
</code>
</pre>

Now, this is something cool I learned the other day. If you move the cursor over the method, you can use the following Vim command to generate the empty test boilerplate. It’s amazing
```
:GoTests
```
Note the plural of the command: “Tests” not “Test”. Singular “Test” will just run the (non-existent) unit tests.

So, let’s see what this magic gave us. Navigate to the test file (`:GoAlt`)
```
package path

import "testing"

func Test_urlSubstitutor_ToRegex(t *testing.T) {
    type args struct {
        swaggerPath string
    }
    tests := []struct {
        name string
        subs urlSubstitutor
        args args
        want string
    }{
        // TODO: Add test cases.
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            subs := urlSubstitutor{}
            if got := subs.ToRegex(tt.args.swaggerPath); got != tt.want {
                t.Errorf("urlSubstitutor.ToRegex() = %v, want %v", got, tt.want)
            }
        })
    }
}
```
Neat!

So, let’s create our first test. This will just be a simple case where there aren’t any parameters in the path to substitute. The first thing we need to do is add an empty struct in the collection of test struct. Then, with the cursor in the empty struct, we do
```
:GoFillStruct
```

The output of this command is as close to magic as I think I’ve ever seen in my life:

<pre>
<code>
package path

import "testing"

func Test_urlSubstitutor_ToRegex(t *testing.T) {
    type args struct {
        swaggerPath string
    }
    tests := []struct {
        name string
        subs urlSubstitutor
        args args
        want string
    }{
        <b>{
            name: "",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "",
            },
            want: "",
        },</b>
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            subs := urlSubstitutor{}
            if got := subs.ToRegex(tt.args.swaggerPath); got != tt.want {
                t.Errorf("urlSubstitutor.ToRegex() = %v, want %v", got, tt.want)
            }
        })
    }
}
</code>
</pre>

Now, we just need to fill in the stubbed fields. Our first test is a path with no substitution so, let’s take a stab at it:

```
        {
            name: "No params",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/path",
            },
            want: "/some/path",
        },
```

We run the tests `:GoTest`and (surprise surprise) the test fails because we don’t have any logic in the substitution method. Let’s add just the bare minimum to get the test to pass

<pre>
<code>
package path

type URLSubstitutor interface {
    ToRegex(swaggerPath string) string
}

type urlSubstitutor struct{}

func (subs urlSubstitutor) ToRegex(swaggerPath string) string {
<b>    return swaggerPath</b>
}
</code>
</pre>

### Something more reasonable
We run our tests and they pass! Now, clearly what we have is insufficient but, let’s see how. We go back to the tests `:GoAlt` , add another empty struct, then fill it `:GoFillStruct`. We’ll fill in the stubbed struct with a path with one substitution.

<pre>
<code>
package path

import "testing"

func Test_urlSubstitutor_ToRegex(t *testing.T) {
    type args struct {
        swaggerPath string
    }
    tests := []struct {
        name string
        subs urlSubstitutor
        args args
        want string
    }{
        {
            name: "No params",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/path",
            },
            want: "/some/path",
        },
<b>        {
            name: "Single param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path",
            },
            want: `/some/[^/\s]+/path`,
        },</b>
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            subs := urlSubstitutor{}
            if got := subs.ToRegex(tt.args.swaggerPath); got != tt.want {
                t.Errorf("urlSubstitutor.ToRegex() = %v, want %v", got, tt.want)
            }
        })
    }
}
</code>
</pre>

We run our tests and they fail. No big deal, let’s improve the logic of our method. We want to replace anything inside a pair of curly brackets with `[^/\s]+`. That is, one or more characters that aren’t a slash or whitespace. Now, I know that that’s a much more broad matcher than what would be a valid url, but in a a later part of the project we’ll be more complex validation, this is just for a broad request match.

Anyhow, to do this we’ll need to call the `ReplaceAllString`method, and substitute anything that lives between two curly brackets with our matcher regex. Something like this:

<pre>
<code>
package path

<b>import (
    "regexp"
)</b>

type URLSubstitutor interface {
    ToRegex(swaggerPath string) string
}

type urlSubstitutor struct{}

func (subs urlSubstitutor) ToRegex(swaggerPath string) string {
<b>    re := regexp.MustCompile(`{(.*?)}`)
    return re.ReplaceAllString(swaggerPath, `[^/\s]+`)</b>
}
</code>
</pre>

Looks promising, but let’s run our tests....success! Now, let’s add more test cases. First, a path with 2 parameters.

<pre>
<code>
package path

import "testing"

func Test_urlSubstitutor_ToRegex(t *testing.T) {
    type args struct {
        swaggerPath string
    }
    tests := []struct {
        name string
        subs urlSubstitutor
        args args
        want string
    }{
        {
            name: "No params",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/path",
            },
            want: "/some/path",
        },
        {
            name: "Single param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path",
            },
            want: `/some/[^/\s]+/path`,
        },
        {
<b>            name: "Double param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path/{otherparam}",
            },
            want: `/some/[^/\s]+/path/[^/\s]+`,
        },</b>
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            subs := urlSubstitutor{}
            if got := subs.ToRegex(tt.args.swaggerPath); got != tt.want {
                t.Errorf("urlSubstitutor.ToRegex() = %v, want %v", got, tt.want)
            }
        })
    }
}
</code>
</pre>

Looks like that passes. How about 3 parameters.

<pre>
<code>
package path

import "testing"

func Test_urlSubstitutor_ToRegex(t *testing.T) {
    type args struct {
        swaggerPath string
    }
    tests := []struct {
        name string
        subs urlSubstitutor
        args args
        want string
    }{
        {
            name: "No params",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/path",
            },
            want: "/some/path",
        },
        {
            name: "Single param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path",
            },
            want: `/some/[^/\s]+/path`,
        },
        {
            name: "Double param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path/{otherparam}",
            },
            want: `/some/[^/\s]+/path/[^/\s]+`,
        },
        {
<b>            name: "Triple param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path/{otherparam}/sure/{one_more_param}",
            },
            want: `/some/[^/\s]+/path/[^/\s]+/sure/[^/\s]+`,
        },</b>
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            subs := urlSubstitutor{}
            if got := subs.ToRegex(tt.args.swaggerPath); got != tt.want {
                t.Errorf("urlSubstitutor.ToRegex() = %v, want %v", got, tt.want)
            }
        })
    }
}
</code>
</pre>

### Negative cases
Still good. Let’s add some negative cases. Now, even if we get an invalidly formatted path string, our substitution method should t return an error. It should just not find a match in the regex substitution. Let’s add 2 new test cases: one missing a closing curly bracket, and one missing the opening curly bracket. In these cases we should expect to get the unaltered path back.

<pre>
<code>
package path

import "testing"

func Test_urlSubstitutor_ToRegex(t *testing.T) {
    type args struct {
        swaggerPath string
    }
    tests := []struct {
        name string
        subs urlSubstitutor
        args args
        want string
    }{
        {
            name: "No params",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/path",
            },
            want: "/some/path",
        },
        {
            name: "Single param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path",
            },
            want: `/some/[^/\s]+/path`,
        },
        {
            name: "Double param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path/{otherparam}",
            },
            want: `/some/[^/\s]+/path/[^/\s]+`,
        },
        {
            name: "Triple param",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param}/path/{otherparam}/sure/{one_more_param}",
            },
            want: `/some/[^/\s]+/path/[^/\s]+/sure/[^/\s]+`,
        },
<b>        {
            name: "Missing closing bracket",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/{param/path",
            },
            want: "/some/{param/path",
        },
        {
            name: "Missing opening bracket",
            subs: urlSubstitutor{},
            args: args{
                swaggerPath: "/some/param}/path",
            },
            want: "/some/param}/path",
        },</b>
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            subs := urlSubstitutor{}
            if got := subs.ToRegex(tt.args.swaggerPath); got != tt.want {
                t.Errorf("urlSubstitutor.ToRegex() = %v, want %v", got, tt.want)
            }
        })
    }
}
</code>
</pre>

And we do.

So there we have it. Once we get our environment setup and hav a good idea of what we want to, test driven development in Go is relatively easy. I hope you found this helpful. In a future post we’ll expand on this by introducing mocking and error cases. Stay tuned!

If you're interested in the project I mentioned above (that this tutorial is writing code for), feel free to [check it out](https://github.com/Simulalex/open-api-sandbox). It's probably pretty sparse at the moment and more _idea_ than _project_, but feel free to contribute, critize, encourage, or dissuade as you feel compelled.
