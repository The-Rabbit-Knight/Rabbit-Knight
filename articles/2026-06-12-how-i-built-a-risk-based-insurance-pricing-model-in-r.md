---
title: "How I Built a Risk-Based Insurance Pricing Model in R"
date: 2026-06-12
medium_url: https://medium.com/@rabbit_knight/how-i-built-a-risk-based-insurance-pricing-model-in-r-057374c3386f
---

> **Original Published Article on Medium:** [How I Built a Risk-Based Insurance Pricing Model in R](https://medium.com/@rabbit_knight/how-i-built-a-risk-based-insurance-pricing-model-in-r-057374c3386f)

### How I Built a Risk-Based Insurance Pricing Model in R

![Article Figure](https://cdn-images-1.medium.com/max/800/1*aT9Ufwmwv08tLR9rlLVD4A.png)

A field report on the messy, satisfying work of turning four mismatched government datasets into an explainable model that actually tells a business what to do.

Most data science write-ups start at the fun part. The model. The accuracy number. The chart that goes up and to the right. This one starts where the real work always starts: with two tables that refused to talk to each other.

I was handed a deceptively simple brief. An insurer wanted to predict an insurance claim score, essentially, how much risk a given neighborhood carries, using nothing but the social, economic, and criminal fingerprint of the area around a customer. Get it right and you can price competitively where risk is low and protect your margins where risk is high. Get it wrong in either direction and you bleed money. The data to solve it existed. It just didn’t fit together. And making it fit turned out to be the most interesting engineering problem in the entire project, more interesting, honestly, than the regression that came after.

Let me walk you through it.

#### The problem nobody warns you about: when your data lives on different maps

Here’s what I was working with:

- Claim scores — the target. 680 observations, recorded at postcode-sector level. - Crime data — 715 observations, also at postcode-sector level. - Population and socioeconomics — the big one. 66 variables, 1,106 observations

Two of those three are at the sector level. Merging them is a Tuesday-afternoon job. The third is the trap.

Postcode sectors and electoral wards are two completely different ways of slicing up the same city. They were drawn by different people for different reasons: one for mail delivery, one for elections, and they do not nest. One sector spills across several wards. One ward bleeds into several sectors. There is no key. There is no lookup table. You cannot simply JOIN.

If you ignore this and force a naive join, you silently corrupt every demographic variable in your model. The accuracy number still prints. The model still “works.” It’s just quietly wrong, and you’ll never know.

So the first real decision of the project was: I’m not joining these tables. I’m going to rebuild the population data from the ward grid onto the sector grid. And to do that, I needed a fourth dataset.

#### Building a bridge out of 319,385 postcodes

I pulled in a complete public list of postcode units — 319,385 of them. Each row ties a tiny cluster of addresses to both its sector and its ward. That overlap is the bridge. If I could count how postcode units are distributed across the sector-ward intersections, I could measure exactly how much of each ward belongs to each sector. But here’s the subtlety that separates a correct solution from a plausible-looking wrong one: You cannot rescale a count and an average the same way.

Think about it. If a ward is split across three sectors, its population should be divided up, a third here, a third there. But its population density should not be divided. Density is a rate. A third of a ward is still just as dense as the whole ward. So I derived two different scaling coefficients.

For absolute values (population, number of households, area), I scale by the ward’s contribution to the sector:

```
K_w = (postcode units in the sector-ward intersection) / (postcode units in the ward)
P_sector_j = Σ ( P_ward_i × K_w[i,j] )
```

For rates and averages (density, percentages, medians), I scale by what fraction of the sector a given ward occupies:

```
K_s = (postcode units in the sector-ward intersection) / (postcode units in the sector)
D_sector_j = Σ ( K_s[i,j] × D_ward_i )
```

Sector population is the sum of intersection populations; express each intersection’s population through the ward’s density times the intersection’s area; divide by sector area; and the area ratios collapse into K_s. A count gets K_w. A rate gets K_s.

In R, the whole bridge came down to three rounds of aggregation on the postcode table: counting units per intersection, per sector, and per ward, then merging those coefficients back onto the population data, applying the right multiplier to each column based on whether it was absolute or relative, and finally aggregating up to sector level

![Article Figure](https://cdn-images-1.medium.com/max/800/1*QmSZltFQQPxeVtgAqMl8Og.png)

#### Cleaning house: missing values, and knowing when not to impute

With the geography solved, the data still wasn’t clean. The population table had a nasty pattern: 230 rows had nothing but a ward code and a postcode district; every other column was missing. A row that is 95% empty cannot be imputed honestly. There’s nothing to impute from. So I made a rule and stuck to it: if a row is missing more than 20% of its variables, it’s gone. Trying to reconstruct it would be inventing data, not estimating it.

The remaining gaps were a different story, the kind of thing where the surrounding variables genuinely carry information about the missing one. For those, I used MICE (Multivariate Imputation by Chained Equations), which treats imputation as its own little modeling layer, so it predicts each missing value from the other variables rather than just slapping a mean on it.

The lesson I keep relearning: imputation is a tool, not a reflex. Knowing which rows deserve imputation is more important than the imputation method itself. One more small thing that saved me time was that I loaded every CSV with as.is = TRUE to keep postcode fields as strings.

#### Feature engineering: where domain thinking beats more columns

By now I had a clean master table at the sector level. The temptation at this point is to throw all 85 columns at a regression and call it modeling.I did exactly that first, on purpose, to build a baseline villain to beat.

R² of 0.65 looks great until you read the coefficient table: almost every single predictor is statistically insignificant. The model “explains” 65% of the variance while being unable to tell you which variables matter. It’s a black box made of noise.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*PajAblMvXJq63GhISHt5ig.png)

So instead of more columns, I built smarter ones. The pattern I leaned on throughout was the weighted composite: collapsing several related raw inputs into a single, more meaningful index, then refining the weights against model performance:

```
# Recombine correlated raw inputs into one weighted index
composite_index 
<-
 w1 
*
 feature_a 
+
 w2 
*
 feature_b 
+
 w3 
*
 feature_c
# A ratio captures behaviour that no single raw column can
behavioural_ratio 
<-
 numerator 
/
 denominator
# A normalised blend, weighted by domain constants
blended_metric 
<-
 
(
p1 
*
 c1 
+
 p2 
*
 c2 
+
 p3 
*
 c3
)
 
/
 scaling_value
```

Three techniques, one idea: replace many noisy, collinear columns with a handful of derived metrics that each encode a clear hypothesis. Some were weighted sums of related groups; some were ratios that turned two columns into one interpretable signal; one was a normalized blend weighted by real-world domain constants. None of them existed in any source file; they’re domain reasoning expressed as arithmetic, and each one folded several correlated inputs into a single term the model could actually use.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*1WaK7Fv4WyDK4nY1trmZxA.png)

#### Teaching a machine to normalize data by itself

Linear regression has assumptions, and the data violated most of them. Predictor after predictor came back skewed. Bell curves, these were not.

The textbook move is to transform each variable by hand: log this one, square-root that one, eyeballing histograms until they look symmetric. With 85 variables, eyeballing is a recipe for inconsistency and a very long afternoon. So I built a small engine to do it for me. The core is a dispatcher that can apply any of eight transformations: none, log, square root, reciprocal, square, logit, Box-Cox, Yeo-Johnson:

```
universal_standardize <- function(v, method) {
  switch(method,
    nothing = v,
    log     = log(v + 1.01),
    sqrt    = sqrt(v + 1),
    reverse = 1 / (v + 1),
    square  = v^2,
    logit   = { n <- (v - min(v)) / (max(v) - min(v));
                log((n + 0.01) / (1.01 - n)) },
    boxcox     = predict(preProcess(data.frame(x = v + 0.01),
                          method = "BoxCox"), data.frame(x = v))[, 1],
    yeojohnson = predict(preProcess(data.frame(x = v + 0.01),
                          method = "YeoJohnson"), data.frame(x = v))[, 1]
  )
}
```

Then I wrapped it in a loop that, for every predictor, tries all eight transformations, runs a Shapiro-Wilk normality test on each result, and reports the winner: the transformation that gets the variable closest to normal. A second version of the engine ranks transformations not by normality but by correlation with the target, so for each variable I could see both “what makes this normal” and “what makes this predictive” side by side, and choose with my eyes open.

The output is a clean recipe: square-root this predictor, Yeo-Johnson that one, Box-Cox another, leave a fourth untouched. The machine does the grinding; I make the calls.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*pJIkHB4HVIfjzbKwdmT_2g.png)

#### Outliers: find the troublemakers, then cap them gently

Influential outliers can hijack a regression: a handful of extreme rows dragging the whole fit toward themselves. I found them with Cook’s distance, which measures how much the model’s predictions would change if you deleted each individual row. The rule of thumb I used: anything more than four times the mean Cook’s distance is suspect.

```
cooksd_train
=
cooks.distance
(
model_Train_Norm
)
plot
(
cooksd_train
,
 pch
=
"*"
,
 cex
=
2
,
 main
=
"Influential obs. by Cook's
distance. Train Data"
)
abline
(
h 
=
 
4
*
mean
(
cooksd_train
,
 na.rm
=
TRUE
)
,
 col
=
"darkorchid"
)
text
(
x
=
1
:
length
(
cooksd_train
)
+
1
,
 y
=
cooksd_train
,
labels
=
ifelse
(
cooksd_train
>
4
*
mean
(
cooksd_train
,
na.rm
=
TRUE
)
,
names
(
cooksd_train
)
,
""
)
,
 col
=
"darkorchid1"
)
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*XpxsKSgvbWISerwCGcqYKQ.png)

That flagged 21 rows in the training set. But I didn’t delete them; deleting data is a last resort, and these were real neighborhoods, not data-entry errors. Instead, I capped them (winsorization): any value past the 1.5×IQR fence gets pulled back to the 5th or 95th percentile.

```
quartile_fix 
<-
 
function
(
df
,
 col
,
 ref
)
 
{
  q    
<-
 quantile
(
ref
[[
col
]
]
,
 
c
(
.25
,
 
.75
)
,
 na.rm 
=
 
TRUE
)
  caps 
<-
 quantile
(
ref
[[
col
]
]
,
 
c
(
.05
,
 
.95
)
,
 na.rm 
=
 
TRUE
)
  H    
<-
 1.5 
*
 IQR
(
ref
[[
col
]
]
,
 na.rm 
=
 
TRUE
)
  df
[[
col
]
]
[
df
[[
col
]
]
 
<
 
(
q
[
1
]
 
-
 H
)
]
 
<-
 caps
[
1
]
  df
[[
col
]
]
[
df
[[
col
]
]
 
>
 
(
q
[
2
]
 
+
 H
)
]
 
<-
 caps
[
2
]
  df
[[
col
]
]
}
```

The payoff was immediate and clean: training R² rose from 0.66 to 0.69. The extremes stopped shouting, and the underlying signal got louder.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*0mu15EJK4_8bJUm4zUoa5A.png)

#### A failure worth keeping in the story

Here’s something I could airbrush out but won’t, because it’s the most instructive moment in the whole project. Right after applying my transformations, before the outlier capping, the model’s test-set RMSE exploded to ~2.8, which is far worse than the untransformed baseline. For a moment, it looked like the whole transformation engine had backfired. It hadn’t. The diagnostics told the real story: a few extreme observations were interacting badly with the transformed scales and detonating the predictions. Once Cook’s distance flagged them, and the winsorization capped them, test RMSE dropped to ~1.08 and then, in the final trimmed models, to the high 0.5s. I keep that detour in because it’s the truth of how modeling actually goes. You don’t walk a straight line from data to model. You build, you watch something break, you read the diagnostics, you fix the real cause. A flawless-looking run usually just means nobody looked hard enough.

#### Four models, one honest competition

I didn’t hunt for a single “best” model. I built four, each a different defensible combination of 6–8 predictors, and let them compete on equal terms. The discipline I held myself to: every coefficient in every final model had to be statistically significant. No passengers.

```
# Each candidate is a different defensible subset of engineered + raw terms
model_1 
<-
 lm
(
target 
~
 feature_a 
+
 feature_b 
+
 composite_1 
+
                composite_2 
+
 composite_3 
+
 feature_c
,
              data 
=
 train
)
model_4 
<-
 lm
(
target 
~
 feature_d 
+
 feature_a 
+
 composite_1 
+
                composite_2 
+
 feature_c 
+
 feature_e 
+
 behavioural_ratio
,
              data 
=
 train
)
# ...and two more, each swapping in a different mix of predictors
```

They came out remarkably close, which is itself a good sign; it means the signal is real and not an artifact of one lucky variable selection:

```
| Model | Predictors | F-statistic | R²    | Adjusted R²|
|-------|------------|-------------|-------|------------|
| 1     | 6          | 91.3        | 0.544 | 0.538      |
| 2     | 6          | 94.2        | 0.552 | 0.546      |
| 3     | 8          | 72.4        | 0.559 | 0.551      |
| 4     | 7          | 84.6        | 0.564 | 0.557      |
```

Every predictor across all four models cleared p < 0.001. We went from a baseline that explained 65% of the variance while telling us nothing to models that explain ~56% while telling us a lot: every term defensible, every term interpretable.

That tradeoff, a little R² for a lot of clarity, is exactly the right one for a business that has to justify its prices.

#### Did it actually generalize? The honest checks

Accuracy on data you trained on is a participation trophy. The real test is data the model has never seen. I held out 20% from the very start with a balanced split:

```
idx   
<-
 createDataPartition
(
master
$
target
,
 p 
=
 
0.8
,
 
list
 
=
 
FALSE
)
train 
<-
 master
[
idx
,
 
]
test  
<-
 master
[
-
idx
,
 
]
```

Then I subjected every model to three independent checks:

1. Train vs. test KPIs** — RMSE, MAE, R² on both sets. A big gap screams overfitting. 2. 4-fold cross-validation with cv.lm — refit on different folds and confirm the slope barely moves. 3. Residual diagnostics — histograms and scatter plots to confirm the errors are centered at zero, roughly constant in spread, and patternless.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*aIZwBs9NKvZWU-QsFDyp0Q.png)

Notice the test-set R² is actually *higher* than train. That’s not a fluke, it’s strong evidence the models aren’t overfit. The cross-validation slopes were near-identical across folds, and the residual histograms were clean bells centered on zero. The models behave the same no matter how you shuffle the data. That’s the whole point.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*WATO89ZfMEU9LtVJKhWk4w.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*503Kc4hpD_S0PkHgJBSg_w.png)

#### What the model actually says and why that matters

A model that can’t explain itself is worthless to a decision-maker. So the final step was turning coefficients back into plain language. This is where regression earns its keep over a black box: every term has a sign, a magnitude, and a significance level, and each one can be read as a sentence. For each surviving predictor I asked three questions: Is the direction (sign) plausible? Is the effect large enough to matter? Is it statistically solid (p < 0.001)?

What came out was a compact, interpretable picture: a handful of drivers pushed the target up, a handful pulled it down, and the signs lined up with common-sense expectations rather than fighting them. That coherence is itself a quality signal; when the surviving coefficients all make sense together, you can trust the model isn’t just fitting noise.

#### What I’d want you to take from this

If you skim back over this, notice what actually consumed the effort. It wasn’t lm(). The regression was the last and easiest 20%.

The 80% that mattered was everything around it: reconciling two incompatible maps into one, deriving the right scaling math for counts versus rates, knowing which missing rows to rescue and which to let go, building reusable engines so the grunt work did itself, and refusing to ship a single coefficient I couldn’t explain to a business stakeholder.

That, to me, is what data science actually is. Not the model. The judgment around the model. The willingness to do the invisible plumbing correctly when a shortcut would have looked just as good on the surface and quietly poisoned everything downstream.

The accuracy number is the easy part. The trustworthy pipeline underneath it is the job.
