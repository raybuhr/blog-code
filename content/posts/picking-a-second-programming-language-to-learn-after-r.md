---
title: "Picking a Second Programming Language to Learn After R"
date: 2018-02-15
---

I’ve been asked by a few different people now what language they should
learn after R. Each time it has turned into a more complex conversation
than what they were expecting. Hence, [the rule of
three](https://twitter.com/drob/status/928447584712253440) and this blog
post.

I really love the R programming language. I haven’t always felt that
way; in fact I really disliked R when I first tried to learn it. For
example:

-   Everything is a vector?!?
-   You can load/import code in multiple ways; `source`, `library`,
    `require`.
-   The base family of
    [`apply`](https://www.datacamp.com/community/tutorials/r-tutorial-apply-family)
    functions behave similarly yet don’t pass parameters consistently
    -   *Side note:* I recommend using the `purrr` library instead, but
        if you must go base try starting with `lapply`).
-   The assignment operator `<-` can go on either side of your code!

These aspects of R are *weird*, sometimes *dangerous*, and sometimes
*useful*. That’s actually a good summary of programming in R in general.

From my perspective, the beauty of R is the rapid development cycle and
built-in data types like matrices and dataframes. With R, we’re used to
being able to write some code, run it, and immediately see the output.
We might expect all programming environments to work like that. When
switching to a “lower level” language, like C, it can be pretty tricky
at first. You’ll want to do common things like check what your data
looks like after loading a file or fix that one function you had a typo
in. Without a REPL (Read-Evaluate-Print Loop), you have to write
basically working code just to compile it, then run the compiled code to
see what you messed up. This sounds super annoying and it often is.
However, if you never go through this process you skip out on what many
other programmers learn by having to deal with it. Once you get used to
this workflow, you start to get used to keeping all the code in your
head to remove that extra waiting time. This has the side-effect of
making you a more *defensive* programmer. You’re basically constantly
checking for errors and trying to handle them gracefully while writing
your code instead of after running it and finding what happened. One
downside of this workflow is that it can make it more annoying to debug
and [pair](https://en.wikipedia.org/wiki/Pair_programming) on since both
(or more) people have to try and keep the code in their head and it’s a
pain to keep going through the modify-compile-run loop to see what small
changes did.

So what language should you learn after R?
------------------------------------------

Well, **it depends on your goals**. Here is my *very opinionated* take
on which language to learn next after R based on a few different
outcomes you might be looking for. I’m assuming R was your first
language, though I stand by my advice here on these languages in general
as well.

1. Your goal is to be a *very easy to employ* data scientist: `python`
2. Your goal is to become a *highly paid data scientist*: `scala`
3. You want to understand how the R code you run actually works on the computer: `C` and a bit of `C++`
4. You want to do more web development OR you want lots of customization/flexibility in your interactive data visualizations: `javascript`
5. You just want my personal recommendation: `go`

### **Python**

-   Python has a deep scientific computing stack and with good support
    for statistics, machine learning, and data visualization.
-   Pyspark is rapidly developing as a requirement for processing data
    out of Spark. While R also has a Spark API, in my experience it is
    much harder to get to work, fewer examples, fewer tutorials, and
    fewer people able to help answer questions online.
    -   *Note:* RStudio has put out
        [`sparklyr`](http://spark.rstudio.com/) which has made
        developing on Spark with R so much more awesome. However, I’ve
        personally experienced Spark SQL to be often less efficient than
        pyspark code (I believe because of the optimizer). I’m not an
        expert on this, it seems to be an underground trend among the
        data engineers I talk to. Either way, pretty much every data
        scientist using Spark is using pyspark or Scala if they have to.
-   Deep learning frameworks. Yes, these also exist in R, but more jobs
    listing deep learning ask for python proficiency. Even the
    `tensorflow` port for R just wraps the python API (which is *totally
    fine*, but if using in production you’ll want to be comfortable in
    both languages).
    -   *Note:* RStudio has recently released
        [`keras`](https://keras.rstudio.com/) for R, which makes
        developing deep learning in R much nicer. It installs
        `tensorflow` and `keras` in a python environment for you and
        wraps the R `tensorflow` library. Ultimately Keras models
        compile to C++ anyway so any model you build can be deployed in
        any environment that can serve models from the backend you
        choose (default is `tensorflow`). However, when you’re getting
        bugs in your R `keras` code, you’ll still be better off knowing
        some python.

### **Scala**

Caveat: This goal may eventually make it *perhaps slightly harder*
finding a job due to being “over-qualified”, though hopefully not
significant difficulty if you actually have **solid** data science
skills.

-   Scala is an interesting language that provides tremendous
    flexability to the programmer.
    -   Scala is a multi-paradigm language supporting Object Oriented
        Programming (OOP), Functional Programming (FP – think
        `tidyverse` libraries where you pipe data between functions),
        and pretty much everything in-between.
    -   It runs on the Java Virtual Machine (JVM) and thus comes with
        libraries available for Java.
    -   While a compiled language, the Scala Build Tool (SBT) is pretty
        nice to new-comers, though can be very confusing in its error
        messages.
    -   While Scala is a strongly typed language, you can still use type
        inference, which helps ease the transition from rarely having to
        tell R what data type your code is.
    -   Scala has good support for concurrency and parallezation of
        code, which can be difficult in R.
-   Two of the most important and widespread “Big Data” applications
    have been written in Scala – Apache Kafka and Apache Spark.
-   Big companies tend to use both of these applications together:
    -   Kafka for distributed logging of web applications;
    -   Spark for processing the data from Kafka and storing more in
        persistent databases.
-   Because big companies have spent a lot of time and money setting up
    their “Big Data”" infrastructure on Kafka and Spark, they need data
    scientists who can use them. You can totally do so from R or Python,
    but Scala often has all the features that APIs to other languages
    might not yet have implemented.

I might have made Scala seem *too good* here. I think it’s a bit
divisive language. The Scala community can be much less friendly than
the R community. I believe this in large part because of how much people
who know a lot about Scala forget people who are new to programming or
new to Scala don’t automatically know. Functional programming is pretty
popular among R programmers, but Scala takes FP to a whole other level.
When searching for help online, you often get bombared with answers that
lead to rabbit-holes of reading to even try to understand. And since
Scala is so flexible, it can often be really hard to follow other
people’s code, especially when they are [being
*crafty*](http://yz.mit.edu/wp/true-scala-complexity/). A Scala
programmer might start off defining classes like OOP, then switch to a
FP style for data manipulation. This practice seems to be pretty common
for R so I think most R programmers can grok it.

### **C and a bit of C++**

-   Many R libraries are wrappers for C and C++ code, so naturally it
    makes sense to learn if you want to work on them or make your own
-   If you know how C works (pointers, data structures, memory
    allocation), then C++ will give you some tools that C by itself
    doesn’t have. The awesome library
    [`Rcpp`](https://cran.r-project.org/web/packages/Rcpp/index.html)
    allows you to easily [inject just a little bit of C++ code in R
    code](http://adv-r.had.co.nz/Rcpp.html) to speed things up.
    -   Similarly, knowing C allows you to use Cython to speed up Python
        code.

### **JavaScript**

-   JS == programming on the web browser
-   Pretty hard to avoid learning some eventually if you program long
    enough
-   JS is pretty easy to get started learning and often JS code looks
    very similar to R code
-   The JS ecosystem is enormous – [475,000 packages right
    now!!!](https://www.npmjs.com/) – which is both a blessing and a
    curse

### **Go**

I’ve started writing [Go](https://golang.org/) over the past few months
and I’ve found it to be a nice in between of Python and C. If all you
know is R, that probably means little to you and thus not why I
typically recommend it to R programmers right away. Writing Go from the
standpoint of someone who is used to writing R code will probably be
pretty difficult at first because of the many adjustments in style.
While anyone can get started very quickly in Go, I’ve found writing
*effective* Go programs means I have to really think through what data
types I’ll need to solve the problem *before* I start writing code. I
often end up declaring my own custom data types through the use of
`structs` and `interfaces`, which makes my programs behave very
consistently and less likely to miss errors. These terms are most likely
foreign constructs to the R programmer, but also maybe help show why it
is a good reason to learn a second language!

Like C, Go is a static typed (have to declare the data type of all
variables), compiled,
[procedural](https://en.wikipedia.org/wiki/Procedural_programming)
language. Having already known Python proficiently, I found Go pretty
easy to pick up. Like C, you need to understand pointers, but otherwise
Go makes it easy to be productive quickly. It has most of the go-to data
structures like Maps (like named lists in R), Arrays (fixed length
vectors), and Slices (variable length vectors) built in. In addition, Go
handles garbage collection for you and has type inference making it a
lot simpler to write than C. Unlike other compiled languages, compiling
a Go program usually only takes 1-2 seconds, which makes comes back
closer to the REPL feel of R or Python. In fact, there are a couple REPL
projects for Go, though they each have their issues (check out
[`gophernotes`](https://github.com/gopherdata/gophernotes) if you want
to try Go in a Jupyter notebook). In addition, the data science space in
Go has been growing pretty rapidly over the past 2-3 years with some
very interesting projects, namely [`gonum`](https://www.gonum.org/). I
recommend trying Go if you want:

-   an easy language to pick up
-   one that gives you ability to utilize static typing
-   quick development cycle similar to a REPL
-   very easy to get started with concurrency (asynchronous programs)
-   awesome command line tools

I know that’s not everyone’s bucket list of reasons to learn a language
(not even mine), but it has been useful to me and does some things
really well like networking, webservers, and building CLI tools.

### Other considerations

There are a lot of really interesting and alternative programming
languages to R and I can’t put them all in a single list. I don’t know
all the languages out there and I don’t have the time to dive deep into
enough to appease everyone. It is very likely that there are alternative
languages not in this list that are probably better recommendations, but
this is my list based on my experiences and insights into the world of
professional data science and I feel confident standing by this list.
Please keep in mind that just because a language is not on this list
doesn’t mean I don’t like it – it just means I don’t recommend it as a
second language after R.

For example, [Rust](https://www.rust-lang.org/en-US/) is a super
interesting language that shares a lot of the benefits over C/C++ that
Go does, while simultaneously providing more guarantees of safety and
flexability of development. I think Rust has great potential to be the
next major programming language for operating systems and situations
where performance is critical. Having read a fair amount of Rust, trying
to figure out the syntax and how Rust programs work is much more
challenging than Go or C. That’s not a bad thing by itself, but in my
personal opinion, learning a second programming language shouldn’t focus
on learning all the things, just learning another style and school of
thought.
