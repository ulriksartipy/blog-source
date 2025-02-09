---
layout: post
title: Extrapolation is tough for trees!
date: 2016-12-10
tag: 
   - R
   - MachineLearning
description: Tree-based predictive analytics methods like random forests and extreme gradient boosting may perform poorly with data that is out of the range of the original training data.
image: /img/0071-four-methods.svg
socialimage: http://ellisp.github.io/img/0071-four-methods.png
category: R
---

## Out-of-sample extrapolation

This post is an offshoot of some simple experiments I made to help clarify my thinking about some machine learning methods.  In this experiment I fit four kinds of model to a super-simple artificial dataset with two columns, x and y; and then try to predict new values of y based on values of x that are outside the original range of y.  Here's the end result:

![four-methods](/img/0071-four-methods.svg)

An obvious limitation of the extreme gradient boosting and random forest methods leaps out of this graph - when predicting y based on values of x that are outside the range of the original training set, they presume y will just be around the highest value of y in the original set.  These tree-based methods (more detail below) basically can't extrapolate the way we'd find most intuitive, whereas linear regression and the neural net do ok in this regard.

## Data and set up
The data was generated by this:
{% highlight R %}
# set up functionality for modelling down the track
library(xgboost)  # extreme gradient boosting
library(nnet)     # neural network
library(ranger)   # for random forests
library(rpart)    # for demo single tree
library(rpart.plot)
library(viridis) # for palette of colours
library(grid)    # for annotations

# sample data - training set
set.seed(134) # for reproducibility
x <- 1:100 + rnorm(100)
y <-   3 + 0.3 * x + rnorm(100)

# extrapolation / test set, has historical data plus some more extreme values
extrap <- data.frame(x = c(x, 1:5 * 10 + 100))
{% endhighlight %}

## The four different modelling methods

The four methods I've used are:

- linear regression estimated with ordinary least squares
- single layer artificial neural network with the `nnet` R package
- extreme gradient boosting with the `xgboost` R package
- random forests with the `ranger` R package (faster and more efficient than the older `randomForest` package, not that it matters with this toy dataset)

All these four methods are now a very standard part of the toolkit for predictive modelling.  Linear regression, $$E(\textbf{y}) =  \textbf{X}\beta$$ is the oldest and arguably the most fundamental statistical model of this sort around.  The other three can be characterised as black box methods in that they don't return a parameterised model that can be expressed as a simple equation. 

Fitting the *linear model* in R is as simple as:

{% highlight R %}
mod_lm <- lm(y ~ x)
{% endhighlight %}

*Neural networks* create one or more hidden layers of machines (one in this case) that transform inputs to outputs.  Each machine could in principle be a miniature parameterised model but the net effect is a very flexible and non-linear transformation of the inputs to the outputs. This is conceptually advanced, but simple to fit in R again with a single line of code.  Note the meta-parameter `size` of the hidden layer, which I've set to 8 after some experimentation (with real life data I'd used cross-validation to test out the effectiveness of different values).

{% highlight R %}
mod_nn <- nnet(y ~ x, size = 8, linout = TRUE)
{% endhighlight %}

*`xgboost`* fits a shallow regression tree to the data, and then additional trees to the residuals, repeating this process until some pre-set number of rounds set by the analyst.  To avoid over-fitting we use cross-validation to determine the best number of rounds.  This is a little more involved, but not much:

{% highlight R %}
# XG boost.  This is a bit more complicated as we need to know how many rounds
# of trees to use.  Best to use cross-validation to estimate this.  Note - 
# I use a maximum depth of 2 for the trees which I identified by trial and error
# with different values of max.depth and cross-validation, not shown
xg_params <- list(objective = "reg:linear", max.depth = 2)
mod_cv <- xgb.cv(label = y, params = xg_params, data = as.matrix(x), nrounds = 40, nfold = 10) # choose nrounds that gives best value of root mean square error on the training set
best_nrounds <- which(mod_cv$test.rmse.mean == min(mod_cv$test.rmse.mean))
mod_xg <- xgboost(label = y, params = xg_params, data = as.matrix(x), nrounds = best_nrounds)
{% endhighlight %}

Then there's the *random forest*.  This is another tree-based method.  It fits multiple regression trees to different row and column subsets of the data (of course, with only one column of explanatory features in our toy dataset, it doesn't need to create different column subsets!), and takes their average.  Doing this with the defaults in `ranger` is simple again (noting that `lm`, `nnet` and `ranger` all use the standard R formula interface, whereas `xgboost` needs the input as a matrix of explanatory features and a vector of 'labels' ie the response variable).

{% highlight R %}
mod_rf <- ranger(y ~ x)
{% endhighlight %}

Finally, to create the graphic from the beginning of the post with the predictions of each of these models using the extrapolation dataset, I create a function to draw the basic graph of the real data (as I'll be doing this four times which makes it worth while encapsulating in a function, to avoid repetitive code).  I call this function once for each graphic, and superimpose the predicted points over the top.

{% highlight R %}
p <- function(title){
   plot(x, y, xlim = c(0, 150), ylim = c(0, 50), pch = 19, cex = 0.6,
        main = title, xlab = "", ylab = "", font.main = 1)
   grid()
}

predshape <- 1

par(mfrow = c(2, 2), bty = "l", mar = c(7, 4, 4, 2) + 0.1)

p("Linear regression")
points(extrap$x, predict(mod_lm, newdata = extrap), col = "red", pch = predshape)

p("Neural network")
points(extrap$x, predict(mod_nn, newdata = extrap), col = "blue", pch = predshape)

p("Extreme gradient boosting")
points(extrap$x, predict(mod_xg, newdata = as.matrix(extrap)), col = "darkgreen", pch = predshape)

p("Random forest")
fc_rf <- predict(mod_rf, data = extrap)
points(extrap$x, fc_rf$predictions, col = "plum3", pch = predshape) 

grid.text(0.5, 0.54, gp = gpar(col = "steelblue"), 
          label = "Tree-based learning methods (like xgboost and random forests)\nhave a particular challenge with out-of-sample extrapolation.")
grid.text(0.5, 0.04, gp = gpar(col = "steelblue"), 
          label = "In all the above plots, the black points are the original training data,\nand coloured circles are predictions.")
{% endhighlight %}


## Tree-based limitations with extrapolation

The limitation of the tree-based methods in extrapolating to an out-of-sample range are obvious when we look at a single tree.  Here'a single regression tree fit to this data with the standard `rpart` R package.  This isn't exactly the sort of tree used by either `xgboost` or `ranger` but illustrates the basic approach.  The tree algorithm uses the values of x to partition the data and allocate an appropriate value of y (this isn't usually done with only one explanatory variable of course, but it makes it simple to see what is going on).  So if x is less than 11, y is predicted to be 4; if x is between 11 and 28 y is 9; etc.  If x is greater than 84, then y is 31.

![tree](/img/0071-tree.svg)

What happens in the single tree is basically repeated by the more sophisticated random forest and the extreme gradient boosting models.  Hence no matter how high a value of x we give them, they predict y to be around 31.  

The implication? Just to bear in mind this limitation of tree-based machine learning methods - they won't handle well new data that is out of the range of the original training data.

Here's the code for fitting and drawing the individual regression tree.

{% highlight R %}
#==============draw an example tree===================
# this is to illustrate the fundamental limitation of tree-based methods
# for out-of-sample extrapolation
tree <- rpart(y ~ x)

par(font.main = 1, col.main = "steelblue")
rpart.plot(tree, digits = 1, 
           box.palette = viridis(10, option = "D", begin = 0.85, end = 0), 
		   shadow.col = "grey65", col = "grey99", 
           main = "Tree-based methods will give upper and lower bounds\nfor predicted values; in this example, the highest possible\npredicted value of y is 31, whenever x>84.")
{% endhighlight %}
