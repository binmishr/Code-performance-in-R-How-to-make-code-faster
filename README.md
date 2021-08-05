# Code-performance-in-R-How-to-make-code-faster


Don’t run the same code multiple times

When using a loop, it sometimes happens that a part of your code does the identical thing in each loop because it is independent of the variable you’re looping over. In this case, you could just do it once before the loop and use the result in each subsequent loop. The same holds for apply structures. In the following example, we filter the dataset with respect to a certain condition within the lapply (filterInside). Since this condition does not change, it can be done before the use of lapply (filterBefore):

microbenchmark(   "filterInside" = {     # For all different species in the iris dataset, do something:     lapply(X = unique(iris$Species), function(spec) {       # Filter: Use only cases with Sepal.Length > 5       dat <- iris[iris$Sepal.Length > 5, ]       # Compute mean Sepal.Width for this species       mean(dat$Sepal.Width[dat$Species == spec])     })   },   "filterBefore" = {     # Filter beforehand:     dat <- iris[iris$Sepal.Length > 5, ]     lapply(X = unique(iris$Species), function(spec) {       mean(dat$Sepal.Width[dat$Species == spec])     })   } )
## Unit: microseconds ## expr             min       lq     mean   median       uq      max neval ## filterInside 379.898 403.1045 490.9101 424.4410 472.4715 3594.538   100 ## filterBefore 258.282 275.5905 324.4593 280.6355 318.4795 2240.069   100

In the first version, the filtering is performed repeatedly for every value of Species. In the second version, it happens only once. If you work with larger datasets or more values of X than just three iris species, this can make a large difference. Apart from filtering, this may apply if you use as.numeric, as.character, or do any other data preparation step.
Avoid appending

Imagine the following: You want to compute multiple results and store them in a vector. You already know the number of the results, i.e., the length of the resulting vector. There are two ways to go about this: Either you start with an empty vector and append every new result (append), or you create a NA vector of the expected length and fill it up step-by-step (fill). As the headline already suggests, the second version is faster:

microbenchmark(   "append" = {     # Create empty vector of zero length     x <- c()     # Append a random number 1000 times     for (i in 1:1000) x <- c(x, rnorm(1))   },   "fill" = {     # Create vector with 1000 NAs     x <- rep(NA, 1000)     # Go through all positions and replace them with a random number     for (i in 1:1000) x[i] <- rnorm(1)   } )
## Unit: milliseconds ## expr        min       lq     mean   median       uq      max neval ## append 3.799159 4.192788 4.829870 4.410594 4.736862 9.440320   100 ## fill   2.869444 3.172670 3.742984 3.367564 3.715332 9.182746   100

Why the difference? Every time you append a new value to x, R needs to make a copy of the old x and allocate space for this copy. This just takes some time. If you create a vector of the target length in advance, all this copying does not happen. A similar situation occurs if you expand a string step-by-step with paste. Instead, you could first create all the parts of the string, and then collapse them together in one single step. In the following example, we create a string of the first ten letters of the alphabet, separated by a comma (the function letters just gives us the letters from A to Z):

microbenchmark(   "append" = {     # Create character with first letter (a)     x <- letters[1]     # Append each letter separately     for (i in letters[2:10]) x <- paste(x, i, sep = ", ")   },   "collapse" = paste(letters[1:10], collapse = ", ") )
## Unit: microseconds ## expr          min       lq       mean    median       uq      max neval ## append   1411.662 1552.267 1860.54328 1706.0015 1991.200 5227.634   100 ## collapse    3.439    4.061    6.51417    5.8275    7.111   36.650   100

The difference is surprising, isn't it?
Vectorize

In the random number example, there would have been an even faster way: vectorization. Instead of using a loop and going through each and every position in the vector, you can just do it all at once. Of course, a loop will still be used somewhere internally. But if you use vectorized functions, these internal loops are implemented in C, which is much faster than R. Whenever possible, take advantage of this. Let us repeat the example and add a vectorized version:

microbenchmark(   "append" = {     x <- c()     for (i in 1:1000) x <- c(x, rnorm(1))   },      "fill" = {     x <- rep(NA, 1000)     for (i in 1:1000) x[i] <- rnorm(1)   },      "vectorize" = rnorm(1000) )
## Unit: microseconds ## expr           min       lq       mean   median        uq       max neval ## append    3877.761 4349.151 5396.08739 4842.093 5401.3660 12775.512   100 ## fill      2838.814 3262.778 4645.08529 3676.416 4344.2275 69052.746   100 ## vectorize   51.645   54.592   62.66464   56.123   61.3565   137.767   100

Vectorization is the clear winner! Of course, this was a very simple example, and vectorization is not always that obvious or may not even be possible. In those cases, you can still take care of creating empty vectors of full length in advance instead of appending results. Further opportunities for vectorization are the functions rowSums, rowMeans, colSums, and colMeans, which compute the row-wise/column-wise sum or mean for a matrix-like object. They are vectorized as well, and hence much faster than using apply, or even looping over the rows or columns.
Use C++

If there is no vectorized solution for your code, it's also possible to translate selected code chunks into C++ on your own. The Rcpp package helps with seamless integration of R and C++. This can make sense for time-consuming loops, for example. It's usually as fast as vectorization. Of course, you need some C++ skills to do this, but you don't need to be an expert.
Save intermediate results

It's as easy as it is helpful: Save your intermediate results in an Rdata file with save(). This can, for example, be prepared data, or a result from a time-consuming model calculation. This way you don't need to repeat the computations more often than necessary.
