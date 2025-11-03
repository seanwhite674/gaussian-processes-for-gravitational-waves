# Gaussian Process Regression for Gravitational Waves
This is a brief summary of my Undergraduate Thesis. The paper is available to read in the repository above.

## Explaining This Repository:

- `1D_ToyProblem` → Introductory GPR examples (learning fundamentals).

- `Final_Model_Code`  
  - `Training_CrossValidated_32GPRs.ipynb` → 32 GPR models trained using 10-fold cross-validation (on 90% of data). 
  - `Ranking_All32_Models.ipynb` → Ranks models based on cross-validation metrics and selects the top 8. 
  - `Testing_Best8Models.ipynb` → Evaluates the 8 best models on the unseen 10% test set.

- `Seán_White_Dissertation.pdf` → My final written report that I submitted as my BSc Thesis. 



## 1. Overview  
This project explores how **Gaussian Process Regression (GPR)** can be used to efficiently predict **waveform mismatches** between analytical gravitational wave (GW) models and **Numerical Relativity (NR)** simulations. NR simulations are the gold standard for generating GW waveforms but are computationally costly. By training GPR models on a limited set of NR-informed mismatches, this project builds a **surrogate model** that generalizes across the binary black hole parameter space, reducing the need for new NR runs while maintaining high predictive accuracy.

### Previous work and Motivation
Recent Bayesian methods (“NR-informed” approaches) use waveform mismatches to determine which analytical GW model is most accurate in each region of parameter space.  However, mismatch computation still depends on expensive NR data.  This thesis proposes a **machine learning alternative** — a Gaussian Process model that predicts mismatches as a smooth function of binary parameters, with quantified uncertainty, allowing faster and more scalable inference.


## 2. Data  
- Mismatch values between analytical model **SEOBNRv5PHM** and NR surrogate **NRSur7dq4**.  
- 250 intrinsic parameters (covering 5 mass ratios and a grid of spin projections), repeated for 4 total masses. (1000 total points) 
- Each mismatch is averaged over 294 extrinsic configurations (such as position in sky).  Again read https://arxiv.org/abs/2409.19404 for further details
- The 8D physical parameter space (two masses + two 3D spin vectors) was reduced to 4D via:  
  - **Total mass:** `M_tot = M₁ + M₂`  
  - **Symmetric mass ratio:** `η = q / (1 + q)²` 'q' is the mass ratio such that 'q = M₂/M₁' 
  - **Spin projections:** `χ∥`, `χ⊥` — components of total spin parallel and perpendicular to the orbital angular momentum

#### Explaining the below Graph:
- Below we have a visualisation of the input data across reduced dimensions. Left: A 3D scatter plot of all
data points for fixed total mass w = 0.25 (Mtot = 37.5M⊙). Centre: A 2D slice of the data at rescaled
symmetric mass ratio z = 0 (q = 0.404), interpolated over spin components. Right: A 1D cut through
the data at x = 0.8 showing variation across y. When using raw data, interpolation is needed between
samples, but GPR provides an analytic model that can be directly evaluated without interpolation.

  <img width="1399" height="351" alt="image" src="https://github.com/user-attachments/assets/49a7cabe-8d73-4f81-9d87-995928046e54" />



## 3. Methodology  
- Below shows a flow-chart of the process followed building and testing the models:

<img width="807" height="460" alt="image" src="https://github.com/user-attachments/assets/6d13a6e1-6a79-4fa1-885a-45f44c06a55c" />





## 4. Gaussian Process Framework  
- Built GP priors and posteriors for both **homoscedastic** (constant noise) and **heteroscedastic** (input-dependent noise) assumptions.  
- Explored multiple kernel types, their shapes and properties are illustrated below
  
#### Explaining the below Graph:
- Visual comparison of common kernel functions and their effect on Gaussian process priors.
Each column shows the kernel shape k(x, x′), samples from the corresponding GP prior, and a summary
of the structure it imposes. All kernels were evaluated using a lengthscale parameter ℓ = 1 (except where
noted). For the Matern kernel, ν = 0.5; Laplace kernel, γ = 6; Rational Quadratic kernel, α = 0.25; and
Periodic kernel, period p = 2. Detailed formulas for each kernel are provided in the appendix 6.2.
Note: In practice, we scale each kernel by a signal variance hyperparameter σ2f, which governs the overall
vertical variation in the function. This scaling is applied consistently across all kernel types and accounts
for the amplitude of the wave.

<img width="667" height="448" alt="image" src="https://github.com/user-attachments/assets/9dcedf2c-b67c-47d1-b3b2-5812d0d7a966" />
 

## 5. Model Training & Evaluation  
- Tested **32 GPR configurations** (different kernel + noise setups).  
- Used **10-fold cross-validation** on 90% of data and a 10% hold-out test set.
- Cross-validated across six metrics:
  - RMSE, MAE, FOM, R², Adjusted R², Pearson correlation.

#### Explaining the below Graph:
- Left: Heatmap showing each model’s rank across the evaluation metrics. The x-axis lists
models according to their ranking in the Table beneath the graph, and the y-axis shows the metrics.
The color bar represents rank (blue indicates better rank, red indicates worse). Middle: Dendrogram showing hierarchical
clustering of metrics based on how similarly they rank models. The vertical axis denotes correlation
distance—smaller values indicate higher agreement between metric rankings. Right: Scatter plot of all
models with R2 on the x-axis, FOM on the y-axis, and MAE represented by the color of each point. The best 8 models form an "optimal"
cluster and we differentiate this from testing on the final test data.
  
<img width="1627" height="491" alt="image" src="https://github.com/user-attachments/assets/3075ca6d-66d0-431b-869d-f74a225ba2d4" />
<img width="759" height="721" alt="image" src="https://github.com/user-attachments/assets/8e97b1a3-86d0-436c-8db2-79c3b8b002e2" />



## 6. Final Model  
The best-performing model was a **heteroscedastic additive GPR** with:

`k(x, x') = σ²_f₁ · k_RBF(x, x') + σ²_f₂ · k_Matern(x, x')`

- The RBF kernel captured global smooth structure.  
- The Matern kernel captured local, noise-like variations.  

<img width="1123" height="245" alt="image" src="https://github.com/user-attachments/assets/6ca38e55-0928-405b-8f84-8289b7923a76" />


## 7. Uncertainty Quantification  
- Used **MCMC** to build a posterior distribution over kernel hyperparameters.  
- The below Graph shows the uncertainty associated with each hyper-parameter. A single tall peak indicates less uncertainty. A wider peak or two peaks indicates much more uncertainty around the optimal hyperparameters.

#### Explaining the below Graph:
- The MCMC results showed clear differences across the model parameters. ℓ₅ had a sharply peaked posterior, indicating it was well constrained, while ℓ₆ was also small, showing it captured much of the local variation in the spin-related dimensions. Parameters like σ²_f₂, ℓ₁, and ℓ₅ were tightly constrained with narrow peaks, meaning their values were stable and confident. In contrast, σ²_f₁ and ℓ₈ showed broad or bimodal posteriors, suggesting uncertainty about how strongly the smooth RBF kernel should dominate over the locally varying Matern kernel when fitting regions of different noise levels.
 
<img width="1431" height="713" alt="image" src="https://github.com/user-attachments/assets/431d3830-4cd0-48fc-a7f6-1b5eea9e459d" />



