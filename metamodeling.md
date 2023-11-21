# Notes on metamodeling for GPCR efficacy


## notes on Raveh et al.

1. "Given the input models", metamodeling proceeds in three steps. 
here, the models are 'final' model with well defined parameters? 
depending of which parameters are assumed fixed and which are not in 
the "surrogate probabilistic models" into which the input models are converted. 

2. the three steps are: 

 - conversion of the input models into surrogate probabilistic models
 - coupling of these surrogate models through subsets of statistically related variables
 - backpropagation to update the original input models by computing the PDFs of 
  free parameters for each input model in the context of all other input models

3. *surrogate probabilistic models* a surrogate model specifies a PDF over some input model variables and any additional variables deemed necessary. This PDF encodes model uncertainty and statistical dependencies among its variables. Model uncertainty arises from insufficient information, imperfect modeling, and/or stochasticity of the system.

we need a PDF. 
for instance, we could define it via a probabilistic graphical model (PGM) 

$$ {\rm PGM}  \supset  {\rm BN} \supset {\rm dyn BN} \supset {\rm HMM} $$




## available models


## defining and sampling BN with 

[tutorial of simple metamodeling](github.com/tanmoy7989/bayesian_metamodeling_tutorial)
this uses [pyMC](https://www.pymc.io/welcome.html) for sampling.


