# Two-part-model

```
import rpy2.robjects as robjects
import rpy2.robjects.packages as rpackages
from rpy2.robjects.vectors import StrVector
#installed needed R package#
packageNames = ('pscl', 'sandwich','lmtest', 'MASS','AER', 'mhurdle')

if all(rpackages.isinstalled(x) for x in packageNames):

    have_packages = True

else:

   have_packages = False

if not have_packages:

    utils = rpackages.importr('utils')
    utils.chooseCRANmirror(ind=1)

    packnames_to_install = [x for x in packageNames if not rpackages.isinstalled(x)]

    if len(packnames_to_install) > 0:

        utils.install_packages(StrVector(packnames_to_install))
        
# read bdata#
import os
cwd = os.getcwd()
os.chdir('E:\Work\python')


r=robjects.r
De=r["read.table"]('data.csv', sep=",", head=True)
print De

## data exploratary
from rpy2.robjects.packages import importr
base=importr("base")
print base.summary(De)
print base.colnames(De)


import rpy2.robjects.lib.ggplot2 as ggplot2
gp=ggplot2.ggplot(De)
pp = gp + \
     ggplot2.aes_string(x='ofp',fill='factor(health)') + \
     ggplot2.geom_histogram()
print pp



## try one-equation models#
formula = 'ofp ~ hosp + health + numchron + gender + school + privins'

## poisson Poisson regression##
fit = robjects.r.glm(formula=formula, family=robjects.r. poisson(link = "log"), 
                     data=De)
print base. summary(fit)


## Fit quasi-Poisson
modelQuasiPoisson = robjects.r.glm(formula = formula,
                         family  = robjects.r. quasipoisson(link = "log"),
                         data    = De)
print base.summary(modelQuasiPoisson)

## Fit negative binomial
mass=importr("MASS")
modelNB = mass.glm_nb(formula = formula,
                  data= De)
print base.summary(modelNB)


## try hurdle (two-part) model

Interview=r["read.table"]('data1.csv', sep=",", head=True)

print base.summary(Interview)
print base.colnames(Interview)

##plot distribution with hist
gp=ggplot2.ggplot(Interview)
pp = gp + \
     ggplot2.aes_string(x='foodaway',fill='factor(race)') + \
     ggplot2.geom_histogram()

print pp

##plost distribution with density QQ-plot 

gp=ggplot2.ggplot(Interview)
pp1= gp + \
     ggplot2.aes_string(x='foodaway',fill='factor(race)') + \
     ggplot2.geom_density(alpha=0.5)

print pp1

pscl= importr("pscl")
zf=pscl.zeroinfl(robjects.r("ofp ~ hosp + health + numchron + gender + school + privins"),
                      data=De, dist = "poisson")
print base.summary(zf)


## Hurdle model
mhurdle=importr("mhurdle")
model=mhurdle.mhurdle(robjects.r("foodaway ~ size + smsa + age + age2 | linc + linc2"),
                      data=Interview, dist = "n") ##normal distribution
print base.summary(model)

model1=mhurdle.mhurdle(robjects.r("foodaway ~ size + smsa + age + age2 | linc + linc2"),
                      data=Interview, dist = "bc") ## box-cox distribution##
print base.summary(model1)

model2=mhurdle.mhurdle(robjects.r("foodaway ~ size + smsa + age + age2 | linc + linc2"),
                      data=Interview, dist = "ln") ##log normal distribution
print base.summary(model2)

model3=mhurdle.mhurdle(robjects.r("foodaway ~ size + smsa + age + age2 | linc + linc2"),
                      data=Interview, dist = "ln", corr=True) ##log normal distribution and correlation is true
print base.summary(model3)


model4=mhurdle.mhurdle(robjects.r("apparel ~ 0 | linc + linc2 | factor(month) + smsa"),
                      data=Interview, dist = "ln", corr=True) ## third zero generation

print base.summary(model4)


H3D=mhurdle.mhurdle(robjects.r("shows ~ educ+size | linc+linc2|age+age2+smsa"),
                    data=Interview, dis="ln", h2=True,corr=True,method="bhhh")

update=importr("stats4")
H3I=update.update(H3D, corr="FALSE")
H2D=update.update(H3D, h2="FALSE")
S2D=mhurdle.mhurdle(robjects.r("shows ~ educ+size+age+age2+smsa| linc+linc2"),
                    data=Interview, dis="ln", h2=True,corr=True,method="bhhh")
P2D=mhurdle.mhurdle(robjects.r("shows ~ 0| linc+linc2|educ+size+age+age2+smsa"),
                    data=Interview, dis="ln", h2=True,corr=True,method="bhhh")

print base.summary(H3D)

stats=importr("stats")
utils=importr("utils")
print stats.coef(H3D,"h2")## extract the estimated coeff for second part
print stats.coef(H3D,"h1") ## extact the estimated coeff for the 1st part
print stats.coef(base.summary(H3D),"h3") ## extract estimated coeff for the 3rd part
print stats.vcov(H3D,'h3') ##extract the variance-covariance matrix of the main parameters
print stats.logLik(H3D) ## extract likelihood value
print stats.logLik(H3D, naive="TRUE") ## extract the likelihood value for nulll model without vars
print base.head(stats.fitted(H3D))
print utils.head(stats.fitted(H3D)) ## esimtated expected value for P(Y=0) and  E(Y|Y>0)

##m model selection##
print mhurdle.vuongtest(S2D, P2D) ## p2d is better than s2d. non-nested model selection
print mhurdle.vuongtest(H3D, H3I, type="nested") ## compared two nested model. Can also check LR test using other package
```
