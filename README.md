

# Exploratory phylodynamics of early EBOV epidemic in Sierra Leone

In this practical, we will analyse some of the first whole-genome EBOV sequences collected during the Spring/Summer 2014 epidemic in Western Africa. The data are described here: 

* [Rates of evolution in the 2014/2015 West African ebolavirus outbreak](http://epidemic.bio.ed.ac.uk/ebolavirus_evolutionary_rates)

By the way, here is an interesting TED talk by the chief scientist responsible for collecting the data: 

* [P Sabeti: How we'll fight the next deadly virus](https://www.ted.com/talks/pardis_sabeti_how_we_ll_fight_the_next_deadly_virus?language=en)

## Installation and setup
For these analyses, we'll use the `phydynR` and `INLA` packages. If you need to install this on MS Windows, run the installation script like this:
```
source('msc_epi_ebola_installScript.R')
```
On Mac or Linux, we will need to compile the package from source, which will take a bit longer:
```
install.packages('devtools')
require(devtools)
install_github( 'emvolz-phylodynamics/phydynR')
install.packages('INLA', repos="http://www.math.ntnu.no/inla/R/stable")
```

Now we load the package as follows:

```r
suppressPackageStartupMessages( require(phydynR) )
```

## Loading and exploring the data
Let's load the multiple sequence alignment and inspect it:

```r
ebov_algn <- read.dna('ebov_panafr_082014.fasta', format = 'fasta' ) 
ebov_algn
```

```
## 121 DNA sequences in binary format stored in a matrix.
## 
## All sequences of same length: 18961 
## 
## Labels: EBOV|KC242801|deRoover|DRC|1976 EBOV|KC242800|Ilembe|Gabon|2002 EBOV|KC242799|13709|Kikwit_DRC|1995 EBOV|KC242798|1Ikot|Gabon|1996 EBOV|KC242797|1Oba|Gabon|1996 EBOV|KC242796|13625|Kikwit_DRC|1995 ...
## 
## More than 1 million nucleotides: not printing base composition
```

It's always a good idea to visually check your alignment, which is most easily done using an external tool like *seaview*.
Note that this alignment includes many sequences from previous EBOV epidemics in addition to the 2014 West African epidemic, and that many of the older sequences are not whole-genomes. Also note that the sequence labels include both the place and time of sampling, and we will need to extract that information. 

Let's compute genetic and evolutionary distances between sequences. This computes the raw number of character differences between each pair of sequences: 

```r
Draw <- dist.dna( ebov_algn, model = 'raw' , as.matrix = T, pairwise.deletion=TRUE) * ncol(ebov_algn )
```

Note the option `pairwise.deletion=TRUE`, which causes missing data to be handled on a pairwise basis as opposed to masking sites across the entire alignment. We can also take a subset of distances from the Sierra Leone samples:

```r
Draw_sl <- Draw[grepl( 'SierraLeone_G',  rownames(Draw)), grepl( 'SierraLeone_G',  rownames(Draw)) ]
diag(Draw_sl) <- NA # don't count zero distances on diagonal
hist(Draw_sl)
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png) 

Here we see that there is very little diversity in the SL data, with many pairs differing by less than two characters. This is due to the short time frame over which the epidemic spread and over which samples were collected.


## A quick phylogenetic analysis and estimation of evolutionary rates
First, we will compute an evolutionary distance matrix for phylogenetic analysis. We will use the *F84* [nucleotide substition model](https://en.wikipedia.org/wiki/Models_of_DNA_evolution), which is similar to the HKY model that several published studies have found to work well for EBOV. 

```r
D <- dist.dna( ebov_algn, model = 'F84', pairwise.deletion=TRUE)
```

Now computing a neighbor-joining tree is as simple as the following command: 

```r
ebov_nj <- nj( D )
ebov_nj
```

```
## 
## Phylogenetic tree with 121 tips and 119 internal nodes.
## 
## Tip labels:
## 	EBOV|KC242801|deRoover|DRC|1976, EBOV|KC242800|Ilembe|Gabon|2002, EBOV|KC242799|13709|Kikwit_DRC|1995, EBOV|KC242798|1Ikot|Gabon|1996, EBOV|KC242797|1Oba|Gabon|1996, EBOV|KC242796|13625|Kikwit_DRC|1995, ...
## 
## Unrooted; includes branch lengths.
```

Let's plot it: 

```r
plot( ladderize(ebov_nj) , cex = .5) 
```

![plot of chunk unnamed-chunk-7](figure/unnamed-chunk-7-1.png) 

Note that there is no sigfinicance to the location of the root of this tree, and branch lengths show distances in units of substitions per site. Note the large cluster involving sequences from Sierra Leone. 

To fit a molecular clock, we must use information about the time of each sample. 
Let's load the date of sampling for sequence, which is found in the following table:

```r
sampleDates_table <- read.csv( 'ebov_sampleDates.csv' ) 
sampleDates <- setNames( sampleDates_table$year, sampleDates_table[,1]) # we will also need this in vector format
```
N
ote that most of the samples come from the most recent epidemic:

```r
hist( sampleDates )
```

![plot of chunk unnamed-chunk-9](figure/unnamed-chunk-9-1.png) 

Now we can construct a time-aware phylogenetic tree. Let's start by placing the root of the tree on a branch that is likely to have the MRCA of the sample. One way to do this is to use the `rtt` command, which uses root-to-tip regression; this selects the root position to maximise the variance in evolutionary distance explained by the tree. 

```r
reroot_ebov_nj <- rtt( ebov_nj, sampleDates )
plot( ladderize(reroot_ebov_nj) , cex = .5)
```

![plot of chunk unnamed-chunk-10](figure/unnamed-chunk-10-1.png) 

Lets do our own root-to-tip regression using the rerooted tree. This will also give us a rough estimate of the molecular clock rate.


```r
n <- nrow(ebov_algn ) # the sample size 
d2root <- dist.nodes( reroot_ebov_nj )[n+1,1:n] # the distance from the root to each tip 
scatter.smooth ( sampleDates, d2root ) # a scatter plot with local regression line
```

![plot of chunk unnamed-chunk-11](figure/unnamed-chunk-11-1.png) 

Finally,we do a linear regression:

```r
summary( lm( d2root ~ sampleDates ) )
```

```
## 
## Call:
## lm(formula = d2root ~ sampleDates)
## 
## Residuals:
##        Min         1Q     Median         3Q        Max 
## -0.0036370  0.0001515  0.0002889  0.0003352  0.0052707 
## 
## Coefficients:
##               Estimate Std. Error t value Pr(>|t|)    
## (Intercept) -1.621e+00  2.906e-02  -55.78   <2e-16 ***
## sampleDates  8.201e-04  1.446e-05   56.70   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 0.001327 on 119 degrees of freedom
## Multiple R-squared:  0.9643,	Adjusted R-squared:  0.964 
## F-statistic:  3215 on 1 and 119 DF,  p-value: < 2.2e-16
```

Specifically, the molecular clock rate is the slope:

```r
coef( lm( d2root ~ sampleDates ) )['sampleDates']
```

```
##  sampleDates 
## 0.0008200608
```

This value is not very accurate, but it's in the right ball-park. Estimates based on the state-of-the-art [Bayesian methods](http://epidemic.bio.ed.ac.uk/ebolavirus_evolutionary_rates) place the rate at around `.00124` substitions per site per year.



## Estimating times of common ancestry 
To estimate a tree with branch lengths in units of time (and TMRCAs), we will need to use algorithms not (yet) included in R. We'll use a fast and accurate heuristic method known as [least-squares-dating](http://www.atgc-montpellier.fr/LSD/)
We run this from the command line as follows: 
```
lsd -i [input tree file name] -d [sample date file name] -o [output file name] -c -r a -s [sequence length]
```

The `-c` option ensures that the tree has only positive branch lengths, and `-r a` examines all branches in the tree as potential sites for the root. `-s` specifies the sequence length, which we have seen from the alignment. 
I have precomputed the results; you can load the time-scaled tree like so:

```r
lsdtree <- read.tree('ebov_lsdtree.nwk')
plot( ladderize(lsdtree), cex = .6 ) 
```

![plot of chunk unnamed-chunk-14](figure/unnamed-chunk-14-1.png) 

We also want a version of this tree with all samples from outside Sierra Leone excluded; this is because we will estimate dynamics in Sierra Leone, and the models will not fit data from outside of that epidemic. We create the SL subtree like so: 

```r
non_sl_samples <- lsdtree$tip.label[ !grepl( 'SierraLeone_G',  lsdtree$tip.label) ] # samples from outside SL
sl_lsdtree <- drop.tip ( lsdtree, non_sl_samples )  # the SL subtree
sl_lsdtree$edge.length <- pmax( 0, sl_lsdtree$edge.length ) # fix any negative bl's
sl_sampleTimes <- sampleDates[ sl_lsdtree$tip.label ] # the sample times just for the SL tree
plot( ladderize( sl_lsdtree ), cex = .7 )
```

![plot of chunk unnamed-chunk-15](figure/unnamed-chunk-15-1.png) 


## Nonparametric phylodynamic estimation 
We will reconstruct the historical dynamics of Ne(t) using the nonparametric skyride technique. This implementation of the skyride model is due to [Julia Palacios](http://juliapalacios.github.io/) (with some modifications), and another example applied to Lassa virus [can be found here](https://github.com/sdwfrost/exploratory-phylodynamics). 

```r
source('skyride.R') # load skyride model
sk <- calculate.heterochronous.skyride( sl_lsdtree ) # just for the SL subtree
```

```r
head(sk)
```

```
##         time sr.median     sr.lc     sr.uc
## 1 0.01047950  9.234598 36.722180 3.3725008
## 2 0.01195890  2.897100  9.250116 1.0625556
## 3 0.01295890  2.228720  6.555603 0.8590725
## 4 0.01321923  1.987580  5.798279 0.7472791
## 5 0.01406541  1.914037  5.513480 0.7363874
## 6 0.01421923  1.906776  5.577032 0.7035862
```

This generates a table of times and Ne estimates. Let's plot the trajectory and credible intervals:

```r
matplot( max(sl_sampleTimes ) - sk$time # show time axis relative to last sample
 , sk[2:4]
 , type = 'l' , log = 'y', col = 'black' , lty = c(1, 2, 2 )
 , xlab = 'Time', ylab = 'Ne(t)' )
```

![plot of chunk unnamed-chunk-18](figure/unnamed-chunk-18-1.png) 

Note the logarithmic y-axis. We see evidence for early exponential growth following by possible decreasing growth rates.



## Phylodynamic estimates using an SIR model 
Let's fit a simple exponential growth epidemic model of the form
```
d/dt I = beta I - gamma I
```
where beta is the transmission rate and gamma is the removal rate. Note that `R0 = beta / gamma`. 
To do this, we build the model by specifying rate equations for births ('transmissions') and deaths: 

```r
births <- c( I = 'parms$beta * I' )
deaths <- c( I = 'parms$gamma * I' )
ebov_model1 <- build.demographic.process(births=births
  , deaths = deaths
  , parameterNames=c('beta', 'gamma') 
  , rcpp=FALSE # specifies that equations are R-code as opposed to C-code
)
```
We will fix `gamma` based on independent estimates. For example, [this study](http://www.ncbi.nlm.nih.gov/pmc/articles/PMC4235004/) found a serial interval of 15 days, so we will use that value after adjusting rates from days to years:

```r
GAMMA <- 365 / 15
```
In a moment, we will estimate `beta` and an initial conditions parameter. First, lets specify starting conditions, and plot a hypothetical trajectory:

```r
t0 <- 2014 # hypothetical start time of SL epidemic
theta_start <- c( beta = 2 * GAMMA, gamma = GAMMA) # starting conditions for maximum likelihood
I0 <- .1 # initial number infected
show.demographic.process( ebov_model1
 , theta = theta_start
 , x0  = c( I = I0 )
 , t0 = t0
 , t1 = max(sl_sampleTimes)
) 
```

![plot of chunk unnamed-chunk-21](figure/unnamed-chunk-21-1.png) 

Now, it is simple to fit the model by maximum likelihood. *Note that this will take a few minutes.*

```r
# first apply sample times to phylo:
sl_lsdtree <- DatedTree( sl_lsdtree, sampleTimes = sl_sampleTimes, tol = Inf, minEdgeLength = 1e-3 ) 
fit <- optim.colik(sl_lsdtree
  , ebov_model1
  , start = c( theta_start, I=I0 ) # the starting conditions for optimisation
  , est_pars = c( 'beta', 'I') # the parameters to be estimated
  , ic_pars = c('I') # the initial conditions
  , t0 = 2014 # the time of epidemic origin (could also be estimated)
  , parm_lowerBounds = c( I = 0, beta = 0) # lower limits for parameter values (can't have negative rates)
) 
```
Here are the parameter estimates:

```r
fit
```

```
## $par
##         beta            I 
## 6.299768e+01 1.311291e-05 
## 
## $value
## [1] 16.90797
## 
## $counts
## function gradient 
##      479       NA 
## 
## $convergence
## [1] 0
## 
## $message
## NULL
```
And here is the estimate of `R0`:

```r
unname( fit$par['beta'] / GAMMA )
```

```
## [1] 2.588946
```

Let's plot our ML model fit along with the cumulative number of samples in our data through time.

```r
show.demographic.process( ebov_model1
 , theta = c( beta = unname( fit$par['beta']), gamma = GAMMA )
 , x0  = c( I = unname( fit$par['I'] ) )
 , t0 = t0
 , t1 =  max(sl_lsdtree$sampleTimes)
 , log = 'y'
 , xlim   = c( 2014.38, 2014.47) 
 , ylim  = c( 1, 1e3 ) 
) 
points( sort( sl_lsdtree$sampleTimes ), 1:length(sl_sampleTimes ) )
```

![plot of chunk unnamed-chunk-25](figure/unnamed-chunk-25-1.png) 

Note that this estimate is very approximate and we should not expect estimated population size to be very precise. Sources of error include:

* Error in phylogenetic reconstruction. We used simple neighbour joining. A likelihood method would be more accurate.
* Low sample size and low diversity of the sample. This will cause estimates to be imprecise. 
* Error in molecular clock rate estimation and heuristic methods for estimating time-scaled trees (we used least-squares-dating).
* Unmodeled heterogeneity. The birth-death model is very simple and potentially unrealistic; we could also fit SIR, SEIR, or models that include heterogeneity in transmission rates (super-spreading). 

How does this estimate of `R0` compare to other published values based on the early epidemic? 

## What next? 
There are many variations on this analysis:

* What are the effects of assuming the F84 model? What about between-site rate variation? Try different nucleotide substition models: `JC` or `TN93`, and compare `gamma=0` vs `gamma=1`
* What are the effects of the starting conditions for the MLE? Can you find a higher likelihood solution with different starting conditions? 
* What are the effects of the birth-death SI model assumptions? This model can be extended in many ways:
	- SIR model: Also estimate the initial number susceptible; this model can capture declining transmission rates towards the end of the sample collection period
	- SEIR model: Once infected, a host is not immediately infectious; for EBOV, the latent period has been estimated to be around 12 days and the infectious period around 3 days. Fit a multi-stage model with fixed rates `gamma0=365/12` and `gamma1=365/3` in units year^-1. 
