---
title: "Using R as a Production Machine Learning Language (Part I)"
date: 2017-10-17
---

There’s often confusion amoung the data science and machine learning
crowd about the quality of R as a production level language for
deploying predictive models. Understandably, many programmers come from
languages that don’t behave quite like R does and focus on libraries
that R has few of. However, R helps us solve problems of data analysis
much more easily that many other languages and some statistical methods
have only been implemented in R. This post has two main goals:

1.  helping full-time developers better understand and appreciate what R
    is capable of;
2.  helping data scientists use their predictive models in reliable
    business settings.

Specifically, I want to demonstrate how we can take a trained predictive
model (i.e. statistical learning model or machine learning model) and
deploy it to a server as a RESTful API that accepts JSON requests and
returns JSON responses. To do this, we will use the package
[plumber](https://www.rplumber.io/). By the end of the post we’ll have R
code that allows us access a website through the browser or
programmatically in order to get predictions.

![what we’ll have at the end of this blog
post](/images/plumber_firefox_screenshot.png)

*Note: The plumber github repo has a giant warning about breaking
changes coming in version 0.4. This is a GOOD thing! When developing an
API like this, you probably want to run it locally in development and
run it locally behind a serious web server like nginx in production.
This post was created using plumber version 0.4.2 so if you follow along
you should not have any problems with the version updates.*

Why is this necessary?
----------------------

There are many ways predictive models can help an organization such as
forecasting finances to better control budget and spending, recommending
products to customers for increased conversion and/or upselling, or
predicting internet traffic to better prepare infrastructure and reduce
costs. There are also many ways we can architect how we use the model,
each with trade-offs. Often with web applications, getting the model as
close (or preferably inside) of the database will yield best
performance, but unless you have a [database that implements R code as
stored
procedures](https://docs.microsoft.com/en-us/sql/advanced-analytics/tutorials/rtsql-using-r-code-in-transact-sql-quickstart),
that often requires precomputing responses. A RESTful API on the other
hand allows us to remain more flexible by only submitting predictions
upon request, thus enabling the ability to turn predictions on/off or
change models more quickly. Let’s take a more real-world scenario as an
example.

**Our objective: recommend a single item for sale to users on their next
login.**

If we go with the first deployment method, we would train our model on
the historical data available at the time and upload our predictions to
a database table (let’s call it `user_item_recommendations`). This
method requires us to work in *batch* mode. We train our data, submit
our predictions, wait for results to come in, then start all over.
There’s nothing wrong with this approach and it might be right for your
organization, especially if you are very concerned about page load
times. Since the prediction is already calculated and the data sits very
close together (in the same database), the web developers just need to
join the `users` table to our `user_item_recommendations` table by the
`user_id` (`uuid`, `uid`, or whatever you use) to quickly identify what
item to display when loading a custom page for each user upon login. A
potential downside is that this process may require modification of the
user login page code, in which case you might have to wait for developer
time to update that part of the codebase. Another potential downside is
that your recommendations might not be very useful unless they change
frequently, which will almost certainly be detrimental to the database
performance and so will have to run at weird hours in the night and
maybe limited to only update a percentage of your users at once.

The API deployment is able to bypass the downsides of the batch update,
database deployment, at the cost of potentially longer page load times.
The good news is that modern javascript has the ability to load content
*asynchronously* so predictions can come in and load content after most
of the page has already rendered. In this case, we might rethink the
objective and end up not really caring about loading the login web page
with the recommended item, but instead just needing to provide a
prediction when triggered by the login event. That is, when a user
actually completes the login to our website, we could respond in a
number of different manners such as sending a SMS/text message, sending
an email, displaying a pop-up or maybe just banner image in the sidebar.
If you take a peek under the hood in many modern websites, you’ll see
this pattern of call and response often. When loading a page, the
browser sends requests based on the url and cookies you submit,
returning custom content.

The real advantage of using the API method for deploying predictive
models is it allows us to more closely align with the [Unix
Philosophy](https://en.wikipedia.org/wiki/Unix_philosophy). By that, I
mean we can treat each unique predictive model as a separate codebase
that serves *one and only one purpose* (make a prediction based on
provided data). This philosophy helps better isolate our code which
makes it easier to understand (which can be hard enough as is with
modeling!), easier to maintain, easier to implement versioning, and
easier to montior our results.

Getting Started with Plumber on a trained model
-----------------------------------------------

This post assumes you already went through the hard part of data
collection, data cleaning (aka munging, transforming, wrangling, etc.),
model training and model validating. If so, save it as a `.Rds` – a
binary file native to R that allows us to efficiently transfer models
saved in R over a network.

If you haven’t done this yet, I’m going to quickly breeze though
building one on the [Titanic example from
Kaggle](https://www.kaggle.com/c/titanic/data) to predict Survival.
*Note that a smaller version is actually built into R, which you can
load with* `data(Titanic)`.

### Prepping Data

Here we start by reading in the data in csv file format to a variable
called `titanic_data`.

{{< highlight r >}}
library(readr)
titanic_data <- read_csv("plumber_titanic/train.csv")
{{< / highlight >}}


In order to ensure the model we build ends up in useful and consistent,
we’ll need the data that gets passed into it to be clean and consistent.
By that I mean things like factors having the same levels and numeric
values that make sense for what they represent. To accomplish that, we
create a simple function to coerce the raw data into the data we expect.
We’ll then be able to apply this function to our training dataset, as
well as any future incoming data we want to predict on.

{{< highlight r >}}
transform_titantic_data <- function(input_titantic_data) {
  ouput_titantic_data <- data.frame(
    survived = factor(input_titantic_data$Survived, levels = c(0, 1)),
    pclass = factor(input_titantic_data$Pclass, levels = c(1, 2, 3)),
    female = tolower(input_titantic_data$Sex) == "female",
    age = factor(dplyr::if_else(input_titantic_data$Age < 18,
                 "child", "adult", "unknown"), 
                 levels = c("child", "adult", "unknown"))
  )
}

clean_titanic <- transform_titantic_data(titanic_data)
{{< / highlight >}}


### Train Model

When training a machine learning model, it’s important to use validation
techniques otherwise you can’t change it without ruining the assumption
that it will continue to work on new (unseen) data. Here we implement a
very simple technique known as train/test split and train our model on
just the `train_df` split. For the model, we use a logistic regression
to estimate the probability of survival based on the passenger’s age,
ticket class, and perceived gender. In business settings, logistic
regression is really practical because it is fairly easy to train (no
GPUs needed), very fast at prediction (fast response time for our time),
and the results are much easier to explain and understand. Specifically
the logistic regression coefficients can help us determine the impact of
each independent variable on the odds of the outcome (survival).

{{< highlight r >}}
set.seed(42)
training_rows <- sample(1:nrow(clean_titanic),
                        size = floor(0.7*nrow(clean_titanic)))
train_df <- clean_titanic[training_rows, ]
test_df <- clean_titanic[-training_rows, ]

titanic_glm <- glm(survived ~ pclass + female + age, 
                   data = clean_titanic, family = binomial(link = "logit"))
{{< / highlight >}}

### Evaluating Model

There are many ways to evaluate model performance, but since that’s not
the point here and there are plenty of areas to confuse and distract
(i.e. I could rant for a while…), let’s keep it simple here. For our
model, we’re using the type `response` which returns probability as our
prediction. From that, we predict `TRUE` for survival is our predicted
probability is over 50%. We then compare the predicted survival to the
actual survival in our `test_df` split to get a *confusion matrix* and
our predicted accuracy.

{{< highlight r >}}
test_predictions <- predict(
    titanic_glm, 
    newdata = test_df, 
    type = "response"
) >= 0.5
test_actuals <- test_df$survived == 1
accuracy <- table(test_predictions, test_actuals)
print(accuracy)
print(paste0(
    "Accuracy: ", 
    round(100 * sum(diag(accuracy))/sum(accuracy), 2), 
    "%"
))
{{< / highlight >}}

```
                test_actuals
test_predictions FALSE TRUE
           FALSE   147   29
           TRUE     29   63

[1] "Accuracy: 78.36%"
```

As you can see, our model does a reasonable job, but could definitely be
dialed in quite a bit. Let’s just call this version 0.0.1 for now, then
as we deploy new and improved versions of our model we can better
justify the investment into our fancy data science.

### Save model to RDS

In order to reuse this model on another computer or just in another R
session, we want to save it. I *highly recommend* using the built-in,
native to R, efficient binary file format or `.Rds`. In addition to
being efficient, since it only saves an R object instead of the entire R
session environment like `.Rdata`, when you load a `.Rds` file later you
can also save it to a variable, which means we can use the `predict`
function with it just like normal (like we just did in previous code).

{{< highlight r >}}
saveRDS(titanic_glm, file = "plumber_titanic/model.Rds", compress = TRUE)
{{< / highlight >}}

Building Plumber API
--------------------

Finally. The good stuff.

I know it took way too long to get here, but I want to make sure you
understood why this was important and I wanted to make sure people new
to R could follow all the way through.

### Plumber API files

We start off by creating two `.R` files:

1.  A file that will define our API endpoints, i.e. the functions we’ll
    be exposing.
2.  A file that will read our API definitions and start up a simple web
    server with those endpoints available.

According to the `plumber` package documentation, the first file is
conventionally named `plumber.R`. Feel free to do that if that helps you
remember where you save the plumber code or helps you easily find the
API definitions once you build 30 of these. Otherwise, feel free to name
it whatever you want. I like to name my API code relevant to the model
it exposes, in this case `titanic-api.R`, since I might include more
than one file in a single directory. That may be bad practice, though so
*you do you*.

The second file contains your code for routing your API endpoints,
commonly referred to as a `router`. There are many possible options
available to customive your API here including adding custom
serializers, adding error handlers, encrypting cookies, and even the
ability to serve static sites. For our use case, we have the most bare
bones scenario (keep it simple) so I simply named it `server.R`. In that
code, we do just 4 things; load in the `plumber` library, source our API
definitions file, and serve it over port `8000`.

{{< highlight r >}}
library(plumber)

serve_model <- plumb("plumber_titantic/titanic-api.R")
serve_model$run(port = 8000)
{{< / highlight >}}

The other file has the most interesting bits. At the top of the code for
`titanic-api.R`, we load the `plumber` library and then read in our
trained model. Then you’ll notice some constants that describe the
current version of the model (and thus the API) and the
variables/features/inputs required. The reason for these constants are
two fold:

1.  We want to make iterative progress on our model and thus we want to
    know which version is serving predictions once we update it.
    Therefore it’s important to update the `MODEL_VERSION` constant
    eveytime you update your model or this file. It’s *totally fine* to
    have millions of versions, but it’s *totally not cool* to have
    multiple versions out there with the same version number.
2.  We want to help developers use our prediction model as a service.
    This means having a way to help them understand what is necessary
    and what the parameters passed to the API mean. We’ll end up
    creating a very simple landing page for our API based on these
    variables.

{{< highlight r >}}
library(plumber)
model <- readRDS("plumber_titanic/model.Rds")

MODEL_VERSION <- "0.0.1"
VARIABLES <- list(
  pclass = "Pclass = 1, 2, 3 (Ticket Class: 1st, 2nd, 3rd)",
  sex = "Sex = male or female",
  age = "Age = # in years",
  gap = "",
  survival = "Successful submission will results in a calculated Survival Probability from 0 to 1 (Unlikely to More Likely)"
)
{{< / highlight >}}

### First use of Plumber Annotation - Health check endpoint

The first function we’ll write will also be the first endpoint we’ll
create. We’re going to start small by simply return a response for any
request containing a status code and our model version. You’ll see that
all we did was write a simple R function, just like we always do.
However, by simply adding a special type of comment right above our
function, `plumber` will automagically be able to turn our code into an
HTTP API. The details of this are maybe a little *complex*, but not
*complicated*. I encourage you to read up on decorator functions if you
are curious, but I’m going to pass on explaining it for now other than
saying they work by telling the program that your function should get
passed into another function as described in the annotation (special
comment).

In this function, the decorator we applied is `@get /healthcheck` which
tells plumber to expose this function to HTTP GET requests at the url
<a href="http://127.0.0.1:8000/healthcheck" class="uri">http://127.0.0.1:8000/healthcheck</a>.

{{< highlight r >}}
#* @get /healthcheck
health_check <- function() {
  result <- data.frame(
    "input" = "",
    "status" = 200,
    "model_version" = MODEL_VERSION
  )
  
  return(result)
}
{{< / highlight >}}

For the function above, think of it as a way to test whether the API is
actually working. In fact, if you stop right here and save your two
files, then source the `server.R` file it should start a web server at
your localhost over port 8000 that you could go check in the browser and
see if it’s running at
<a href="http://127.0.0.1:8000/healthcheck" class="uri">http://127.0.0.1:8000/healthcheck</a>.
Over time, I think you’ll find having a simple endpoint like this can be
really helpful for debugging whether your model is having problems or if
the API server is having problems.

![health check page](/images/plumber_healthcheck_screenshot.png)

### Landing Page

This is probably unnecessary as all software developers are super smart
and obviously know exactly what data to pass a simple REST API, but just
in case let’s make a simple home page for users accessing our API to
help get them acquainted. You’ll notice here in addition to the `@get`
decorator, we also introduce a second decorator, `@html` that tells
plumber to render responses to this endpoint as `html` content instead
of `json`.

{{< highlight r >}}
#* @get /
#* @html
home <- function() {
  title <- "Titanic Survival API"
  body_intro <-  "Welcome to the Titanic Survival API!"
  body_model <- paste("We are currently serving model version:", MODEL_VERSION)
  body_msg <- paste(
    "To received a prediction on survival probability,", 
    "submit the following variables to the <b>/survival</b> endpoint:",
    sep = "\n")
  body_reqs <- paste(VARIABLES, collapse = "<br>")
  
  result <- paste(
    "<html>",
    "<h1>", title, "</h1>", "<br>",
    "<body>", 
    "<p>", body_intro, "</p>",
    "<p>", body_model, "</p>",
    "<p>", body_msg, "</p>",
    "<p>", body_reqs, "</p>",
    "</body>",
    "</html>",
    collapse = "\n"
  )
  
  return(result)
}
{{< / highlight >}}

The html we generate with this function is pretty basic, but if you are
unfamiliar we are displaying:

-   a page header with the name of the API
-   a short welcome message
-   a sentence stating the current version of the model being served
-   a short explanation of how to use the API and what the response
    means

Sourcing the `server.R` file now should allow you to check it out and
see:

![api landing page](/images/plumber_landing_page_screenshot.png)

### Prediction Endpoint

Before jumping straight into the function to make predictions, let’s
start by implementing the helper function we used to clead the data
before training the model, as well as a helper function to validate that
inputs passed to our API are useful and appropriate.

{{< highlight r >}}
transform_titantic_data <- function(input_titantic_data) {
  ouput_titantic_data <- data.frame(
    pclass = factor(input_titantic_data$Pclass, levels = c(1, 2, 3)),
    female = tolower(input_titantic_data$Sex) == "female",
    age = factor(dplyr::if_else(input_titantic_data$Age < 18,
                 "child", "adult", "unknown"), 
                 levels = c("child", "adult", "unknown"))
  )
}

validate_feature_inputs <- function(age, pclass, sex) {
  age_valid <- (age >= 0 & age < 200 | is.na(age))
  pclass_valid <- (pclass %in% c(1, 2, 3))
  sex_valid <- (sex %in% c("male", "female"))
  tests <- c("Age must be between 0 and 200 or NA", 
             "Pclass must be 1, 2, or 3", 
             "Sex must be either male or female")
  test_results <- c(age_valid, pclass_valid, sex_valid)
  if(!all(test_results)) {
    failed <- which(!test_results)
    return(tests[failed])
  } else {
    return("OK")
  }
}
{{< / highlight >}}

You may notice that the `transform_titantic_data` function is almost
identical, but not quite. Since we are now just making predictions, we
don’t need to format the output variable we trained against since it
won’t be in the request. For the second function, you should notice it
is a series of logical tests against each feature separately, that we
then parse to identify if there are any exceptions we might need to
notify in the response. For example, if the Age submitted is excessive
or negative, we probably want to tell the developer to check their code
instead of just returning a probability.

Now onto the main event.

You can see that we implement the API to the same endpoint of
`/survival` and we allow both GET and POST requests to that same
endpoint. Technically this shouldn’t make a lot of sense as HTTP
requests should roughly map to the database operations in the following
fashion according to [principles of good API
design](https://codeplanet.io/principles-good-restful-api-design/):

    GET (SELECT): Retrieve a specific Resource from the Server, or a listing of Resources.
    POST (CREATE): Create a new Resource on the Server.
    PUT (UPDATE): Update a Resource on the Server, providing the entire Resource.
    PATCH (UPDATE): Update a Resource on the Server, providing only changed attributes.
    DELETE (DELETE): Remove a Resource from the Server.

However, while we really only want to allow users to GET a single
prediction back, but not all servers are ok with this strategy. In
theory, a GET request should be to retrieve a specific thing (which this
is), but allowing different payloads in the GET request can break any
caching the server might provide. So even though we want to accept a GET
request with a payload or pass in the payload as URL query string, we’re
going to do both to make everyone unhappy.

{{< highlight r >}}
#* @post /survival
#* @get /survival
predict_survival <- function(Age=NA, Pclass=NULL, Sex=NULL) {
  age = as.integer(Age)
  pclass = as.integer(Pclass)
  sex = tolower(Sex)
  valid_input <- validate_feature_inputs(age, pclass, sex)
  if (valid_input[1] == "OK") {
    payload <- data.frame(Age=age, Pclass=pclass, Sex=sex)
    clean_data <- transform_titantic_data(payload)
    prediction <- predict(model, clean_data, type = "response")
    result <- list(
      input = list(payload),
      reposnse = list("survival_probability" = prediction,
                      "survival_prediction" = (prediction >= 0.5)
                      ),
      status = 200,
      model_version = MODEL_VERSION)
  } else {
    result <- list(
      input = list(Age = Age, Pclass = Pclass, Sex = Sex),
      response = list(input_error = valid_input),
      status = 400,
      model_version = MODEL_VERSION)
  }

  return(result)
}
{{< / highlight >}}

Compared to the rest of the code above, this function is definitely
doing a lot more. First off, we specifically cast our input variables to
the data types we want. JSON, which is how the request comes in, is not
strongly typed and so we may receive an Age of “22” which would work
fine, but fail if we didn’t convert to an integer in R. Next, we use the
helper function to test and validate the inputs passed in the request
and save the results to a variable named `valid_input`. Then we proceed
down two separate paths.

If the `valid_input` was `OK`, then we join the data passed in the
request into a `data.frame`, clean that `data.frame`, and then run the
`predict` function on the request data using our trained model. Instead
of just returning the prediction directly, we want to return it in a
more standard JSON format with some additional information helpful to us
later. Specifically, we return JSON containing:

-   the input passed to the predict function
-   the prediction as both a probability (0.0 to 1.0) and a
    recommendation (TRUE/FALSE)
-   a status code of `200` since everything worked `OK`
-   the version of the model the prediction came from

![predict success](/images/plumber_survival_prediction_screenshot.png)

Why do we need all that? Well, in addition to making predictions, as
data scientists we want to collect data. We want to log how often the
model is being used. We want to later compare how strongly our
recommendation correlates with other actions. We want to be able to
debug our prediction API if things start to go awry.

If the `valid_input` was not `OK`, we don’t want the API to just break
down. Instead we want it to return useful information, like why it
didn’t work. Therefore when we can’t validate the request data, we
return:

-   the exact input passed to the API (not necessarily the same as what
    would have been passed to the model if had been ok)
-   an array of the errors with the input to give the developer a better
    understanding of why the request failed
-   a status code of `400` which indicates a `Bad Request`
-   the version of the model that made the response

![predict error](/images/plumber_survival_error_screenshot.png)

In addition to being able to use the API from a web browser, you can
also use other tools, like curl through the command line.

![curl api](/images/plumber_curl_screenshot.png)

### Putting it all together

Once you’ve got your code ready, let’s go through this checklist to make
sure everything is in order:

1.  We have our code to build a model isolated from the code to deploy
    the model.
2.  We have our model trained and saved to a file.
3.  We have our code to deploy the model as a RESTful API in two files,
    one defining our API in decorated functions and one to serve that
    first file.
4.  We have versioning for our deployment and make sure to update it
    with each change.
5.  We have error handing in place to ensure bad requests get returned
    with useful information.

Concluding Remarks
------------------

Full source code to build this plumber based API is available at
<https://github.com/raybuhr/plumber-titanic>.

Read through the [Plumber Documentation](https://www.rplumber.io/docs/).
By default, all plumber endpoints return responses as `application/json`
using the `jsonlite::toJSON` function. You can read more about
alternative serializers (i.e. other types of response content) [here in
the plumber
docs](https://www.rplumber.io/docs/rendering-and-output.html#serializers).
In my experience, you may be fine with the default serializers or you
may need to get pretty deep in the weeds depending on how you need to
receive and return results.

Deploy your models somewhere that is easy to find and easy to download
for your servers, such as AWS S3 or Google Cloud Storage. GitHub is
usually not a very good place to store models, unless they are very
small and they benefit from reviewing the text differences (which `.Rds`
does not).

In Part II of this post, we’ll explore:

-   uploading our code to a version control system like GitHub
-   uploading our `.Rds` file to cloud storage
-   spinning up a virtual machine in the cloud to host our API
-   using Docker to *“containerize”* our API allowing us to run multiple
    instances at once
-   using Nginx to serve up those docker instances of this API and act
    as a load balancer
