# Matching


- …
- Note on using inverse probability weights instead of propensity scores
  for matching (*The Effect* p. 300)
- Even though we’re trying to predict the propensity score, we want to
  think about a causal diagram still since we’re trying to close
  backdoor paths (*The Effect* p. 300)

``` python
import pandas as pd
import numpy as np
import statsmodels.formula.api as sm
# The more-popular matching tools in sklearn
# are more geared towards machine learning than statistical inference
from causalinference.causal import CausalModel
from causaldata import black_politicians
br = black_politicians.load_pandas().data

# Get our outcome, treatment, and matching variables as numpy arrays
Y = br['responded'].to_numpy()
D = br['leg_black'].to_numpy()
X = br[['medianhhincom', 'blackpercent', 'leg_democrat']].to_numpy()

# Set up our model
M = CausalModel(Y, D, X)

# Estimate the propensity score using logit
M.est_propensity()

# Trim the score with improved algorithm trim_s to improve balance
M.trim_s()

# If we want to use the scores elsewhere, export them
# (we could have also done this with sm.Logit)
br['ps'] = M.propensity['fitted']

# We can estimate the effect directly (note this uses "doubly robust" methods
# as will be later described, which is why it doesn't match the sm.wls result)
M.est_via_weighting()

print(M.estimates)

# Or we can do our own weighting
br = br.assign(ipw = lambda x: x.leg_black*(1/x.ps) + (1-x.leg_black)*(1/(1-x.ps)))

# Now, use the weights to estimate the effect (this will produce 
# incorrect standard errors unless we bootstrap the whole process,
# as in the doubly robust section later, or the Simulation chapter)
m = sm.wls(formula = 'responded ~ leg_black',
weights = br['ipw'],data = br).fit()

m.summary()
```


    Treatment Effect Estimates: Weighting

                         Est.       S.e.          z      P>|z|      [95% Conf. int.]
    --------------------------------------------------------------------------------
               ATE      0.047      0.080      0.588      0.557     -0.109      0.203

|                   |                  |                     |           |
|-------------------|------------------|---------------------|-----------|
| Dep. Variable:    | responded        | R-squared:          | 0.022     |
| Model:            | WLS              | Adj. R-squared:     | 0.022     |
| Method:           | Least Squares    | F-statistic:        | 128.6     |
| Date:             | Wed, 04 Mar 2026 | Prob (F-statistic): | 1.75e-29  |
| Time:             | 13:55:49         | Log-Likelihood:     | -5768.6   |
| No. Observations: | 5593             | AIC:                | 1.154e+04 |
| Df Residuals:     | 5591             | BIC:                | 1.155e+04 |
| Df Model:         | 1                |                     |           |
| Covariance Type:  | nonrobust        |                     |           |

WLS Regression Results

|           |        |         |        |          |         |         |
|-----------|--------|---------|--------|----------|---------|---------|
|           | coef   | std err | t      | P\>\|t\| | \[0.025 | 0.975\] |
| Intercept | 0.4100 | 0.009   | 43.596 | 0.000    | 0.392   | 0.428   |
| leg_black | 0.1499 | 0.013   | 11.340 | 0.000    | 0.124   | 0.176   |

|                |          |                   |             |
|----------------|----------|-------------------|-------------|
| Omnibus:       | 2184.371 | Durbin-Watson:    | 1.936       |
| Prob(Omnibus): | 0.000    | Jarque-Bera (JB): | 2062976.257 |
| Skew:          | 0.078    | Prob(JB):         | 0.00        |
| Kurtosis:      | 97.087   | Cond. No.         | 2.63        |

<br/><br/>Notes:<br/>[1] Standard Errors assume that the covariance matrix of the errors is correctly specified.

- If omitted variable bias is equivalent to “open back doors” then
  conditional independence is equivalent to “all the back doors are
  closed” (*The Effect* p. 304)
- “The assumption of common support says that there must be
  *substantial* overlap in the distributions of the matching variables
  comapring the treated and control observations. Or in the context of
  propensity scores, there must be substantial overlap in the
  distribution of the propensity score.” (*The Effect* p. 306)
- We assume that the true propensity score is never exactly 0 or 1 to
  avoid dividing by 0 when weighting (*The Effect* p. 307)
- We can “trim” the propensity score and limit the data to the range of
  common support (*The Effect* p. 308)
- “Matching relies on the existence of comparable observations” (*The
  Effect* p. 309)
- Trying to achieve balance in the comparable groups is akin to balance
  requirements for an experimental design
- Again, we run into a problem of getting standard errors or confidence
  intervals and the need to use Bayes or bootstrap methods (*The Effect*
  p. 316)
