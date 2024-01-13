---
title: "Functional Tests are a Superset of Regression Tests. Usually"
date: 2024-01-08
---

When it comes to Software Testing, there are an endless number of labels we can apply to an endless number of ways to test software. 
Regression testing, soak testing, behavioral testing, unit testing, load testing, -ility testing,
functional testing etc. And even though tight definitions can be somewhat controversial, I think they're incredibly important. 
I'm a big proponent of a strong team vocabulary. And well-defined labels for the critical processes and components of a team make communication easier. 
Words are after all, an abstraction on ideas. And just about the only tool we all have to express those ideas. If you can name something, you can reason about it. 
And if your audience understands, you can reason together. And like any other tool, it's important to keep it sharp, oiled, and quickly available when needed. 

But what I'm here to talk about is totally unrelated how much _I_ think you should care about the definitions of words. Today, we're going to discuss the pesticide 
paradox, and how it makes any sense at all to even contemplate throwing away tests you wrote. First, a few definitions:

* **Functional Testing**: A type of testing that validates if a certain feature, module, component, or unit of software behaves according to the functional requirements
* **Regression Testing**: A type of testing that ensures the behavior of a certain feature, module, component, or unit of software has not changed since the last time it was tested. 
* **Pesticide Paradox ([Fifth testing principle in the ISQTB](https://istqb-main-web-prod.s3.amazonaws.com/media/documents/ISTQB_CTFL_Syllabus-v4.0.pdf))** Used as metaphor to describe the phenomena that tests become less effective("wear out"). And most tests reach a point where the cost to maintain them is not worth the value they provide. 

## The Paradox of the Pesticide Paradox
For any group that is newly on their formalized testing journey, or who just can't stomach the idea that _less testing can lead to better outcomes_,
the pesticide paradox sounds like someone drank the pesticide, instead of spraying it. After all, time and effort was spent writing these tests, and now we're just supposed to throw them away? 

Usually at the heart of this push back exists a misconception about the role tests play in the software development lifecycle. Most often, the code gets written 
and manually validated(by the author or some QA person) to meet the requirements of the task. Tests are then created to enshrine these functional requirements in code, and run forever on every branch until 
they're so old the code they test is deleted from the repo. These tests are the classic example of _late regression tests_. And they absolutely have their place 
in every testing portfolio. But writing tests this way misses out on a huge aspect of their value.

Any acolyte of Test Driven Development will tell you the above approach to test writing is tantamount to sacrilege. 

That's because any code that is written should have its first invocation be by suite of tests designed to validate all functional requirements. It should not be validated only after going through
all the effort to integrate it into the overall application. In TDD this manifests most often as Unit tests written first before the code, 
and the code written to make the test compile (known as the "red green cycle"). When done like this, you can immediately validate that the code you just wrote is correct. It also means you can validate 
the code in isolation, before integrating it into the rest of the application. And if changes are required as part of integration, then you have an applicable test suite to provide quick
feedback on those changes. 

## An Example
To illustrate this point, take the following contrived example:

I need a method to extract the password from an encoded string. This string has the format username:password and then base64 encoded.
I wrote the following code to handle that:

```ASP.NET (C#)
public string ExtractPassword(string encodedString)
{
    //The string is base 64 encoded
    var base64EncodedBytes = System.Convert.FromBase64String(base64EncodedData);
    var str = System.Text.Encoding.UTF8.GetString(base64EncodedBytes);
    
    //encoded string will be of the format <username>:<password>
    //Split at the colon and return the last portion
    return str.Split(":").Last();
}
```

I'm reasonably confident that this code works. But to really ensure that it works as expected, I'm going to have to test it. I've heard unit testing is a good idea, 
so I devise to write the following tests:

1. Parameter is an empty string. Should return... an empty string?
1. Parameter is a properly encoded/formatted string. Returns the password
1. Parameter is a base 64 encoded string that contains two colons. eg username:asdf:qwerty. I'm unsure what this should return
1. Parameter is not correctly base 64 encoded. Should also return an empty string?
1. Parameter is a base 64 encoded string that contains 0 colons. Should return an empty string 

Immediately as I was writing these tests I realized something wasn't right. I really didn't know how this method was supposed to handle some of these edge cases,
and after writing these tests I realize that it does not handle these edge cases gracefully at all. 

These are the major takeaways from each test:

1. A return value of empty string in an error case is really not awesome. It doesn't clearly indicate failure. But also an exception is thrown with my current implementation
2. This worked. Which confirms for me the code does in fact work. But happy-path working is clearly not adequate. 
3. The method returned "qwerty", but this scenario should actually return "asdf:qwerty" 
4. This also just throws an exception, but expecting an empty string is still not a great result. 
5. This actually just returned the string passed in.... which seems like completely wrong behavior. 

So with these in mind, I've decided to refactor my code to meet the newly determined acceptance criteria

```ASP.NET (C#)
public bool TryExtractPassword(string encodedString, out string password)
{
    try {
        //The string is base 64 encoded. Will throw an exception if that is not the case
        var base64EncodedBytes = System.Convert.FromBase64String(base64EncodedData);
        var str = System.Text.Encoding.UTF8.GetString(base64EncodedBytes);
    
        //encoded string will be of the format <username>:<password> where the username is alphanumeric, and the password can be any character
        var matches = Regex.Match("([0-9a-zA-Z]+):(.+)");
        
        //The decoded string did not match the format, fail
        if (!matches.Any())
        {
            return false;
        }
        
        //Return the second capture group. There will always be two if the match was successfull 
        password = matches[1];
        return true;
    } catch (Exception e)
    {
        password = null;
        return false;
    }
}
```

With a slight refactor to my existing unit tests, and updating them to assert on the new expectations, I find they all pass and the code quality to 
be dramatically improved. I also have a suite of tests that can be kept as regression tests to ensure this code always meets functional requirements. 

These _early functional tests_ provide a great deal of value. Not only did they inform me on the behavior of my code, they highlighted some serious
gaps in my pre- and post-conditions, as well as my architecture. The original code was bad. Sure it met the primary criteria, but it's the kind of code that bites the next person who has to touch it. 

As you hopefully have seen, adding these tests to the regression suite is a mere byproduct, but not how they contributed their primary value. 
While the tests exemplified here are inexpensive to run and maintain long term, not every test written during this phase of development will require that they be kept long term. 
In fact, at times it may be advantageous to retire certain tests, simplifying the testing suite. Especially if the tests are of poor quality, complex, or require a specialized environment. 

This is the heart of the pesticide paradox. Functional tests might serve a purpose, or be written such that they're ill-suited for regression testing. 
And harnessing this technique can unlock a new world of testing best practices, and double the value provided by your testing suites. 

# The Non-Regression test

As we're all aware, making software production ready takes 10x as much effort as it does to write software for a prototype. 
Regression tests are the production level tests. They must pass consistently, they must be easy to maintain, they must run with a reasonable level of performance. 
But as we also know, some of the most valuable software we've ever written is the prototype. The quickly written script that'll never see beyond your machine, but that solved a problem 
that would have been incredibly difficult to solve otherwise. That's the non-regression test. They don't have to be pretty, nor are they bound by 
the best practice standards of the team. They exist to inform, and validate the code under development. And upon completion, can be promoted to the regression suite if appropriate, 
or retired. 

# Conclusion

I used in my example the quintessential unit test. But these concepts apply to every level of testing. Be it an integration test, performance test, UI test, or even manual testing. 
Identifying opportunities to write non-regression tests, and making the effort to write early functional tests can save time, improve quality, and 
improve the overall development experience. The pesticide paradox is after all something we know well, just applied to the world of testing.  


