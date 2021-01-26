# Robust-Shared-Response-Model
# Introduction 

In the beginning of our fall semester of grad-school, we began working on a Brain Computer Interface Capstone project for the Department of Psychology at the University of Virginia. Brain Computer Interfaces (BCI) use electroencephalography (EEG) data collected from the human brain to predict behavior. This technology will soon be capable of enhancing prosthetic technologies, and perhaps in the far off future lead to even more astonishing and strange human advancements such as artificial memory storage. 

There are, however, many hurdles to successfully implementing a model that predicts human behavior. Since these models interact directly with humans, the stakes of failure are extremely high. If a prosthetic foot malfunctions while driving a car, lives could be lost. Furthermore, these models often take a very long time to train. Brain signal topographies look different for different people when performing the exact same actions. In addition, brains often respond to the same event at different time intervals. 


![Subject 1](images/subj8.JPG)
![Subject 2](images/subj2.JPG)

The above image shows the topography of two different brains responding to the exact same event at the same time. However, the topographies look quite different to the human eye. Because of these differences, current models are usually trained with no prior knowledge. This leads to extensively long train times which are both expensive and not very practical for real world scenarios. We propose that a model initially built for fMRI modeling called the Shared Response Model can be applied to EEG modeling. The idea behind the Shared Response Model is to extract components that are shared across subjects in order to pre-train a model before it is even used on an individual subject. The initial findings from the fMRI model found that using a shared response model both increased the accuracy and reduced the train time of models built for specific people. Naturally, we did not find it to be a far-fetched idea to try to implement this model with a different, albeit, similar type of data. 
	
Recently, a newer version of the shared response model was implemented called the “Robust Shared Response Model”. This model incorporates a new variable that represents what is unique to each individual. In other words, the robust shared response model is capable of distinguishing between what is shared by all participants and what is unique to each participant. The team who implemented the new model found that “RSRM was able to improve the accuracy more than 60% over SRM for the coding of infant fMRI data.” This algorithm includes lots of fun tricks such as solving the Orthogonal Procrustes problem which is named after the mischievous son of Poseidon who wrangles and delimbs his victims so they can fit into his bed.  

<center>

![Yikes](images/Bed-oF-pRocrustes.jpeg)
</center>

This blog post will go into detail about how the robust shared response model is working by implementing the code in R. This post will not include using EEG for the RSRM. That, after all, is for our next semester in grad school. We did, however, find the RSRM algorithm to be quite beautiful so we wanted to share how it works with the student teacher community. Many readers may find that the algorithm is multifunctional and can be used across domains. In the end we will show how the RSRM is capable of extracting components from multiple subjects with a couple shared sine-waves, with noise added for realism. 

# Overview of RSRM


Let $N$ be the number of subjects, $v$ the number of features, $k$ the number of latent components, and $t$ the number of time-points.

The following expression is the primary equation for the Robust Shared-Response model

\begin{equation}\tag{1}
\mathbf{X}^{(i)} = \mathbf{W}^{(i)}\mathbf{R} + \mathbf{S}^{(i)} + \mathbf{E}^{(i)},\ i = 1 \dots N
\end{equation}

where $(i)$ is the indexer for each individual subject.

We can represent the dimensionality of our equation with this image:

![These are the matrices that represent the Robust Shared Response Model](images/rsrm_diagram.png)

### **Variables**

- $\mathbf{X}^{(i)} \in \mathbb{R}^{v_i×t}$ is the data matrix, 

- $\mathbf{W}^{(i)} \in \mathbb{R}^{v_i×k}$ is the matrix mapping from the observed subject space to the shared latent space, 

- $\mathbf{R} \in \mathbb{R}^{k×t}$ is the shared-response matrix, 

- $\mathbf{S}^{(i)} \in \mathbb{R}^{v_i×t}$ is the non-shared matrix unique to each individual subject, and 

- $\mathbf{E}^{(i)} \in \mathbb{R}^{v_i×t}$ is an additive noise matrix specific for each subject.

### **Additional Parameters**
**Lambda** ($\lambda$)

The shrinkage parameter $\lambda$ balances how much is thought to be shared  ($\mathbf{R}$) by each subject and how much is unique to each subject ($\mathbf{S}^{(i)}$). As $\lambda \rightarrow \infty$, the model is equivalent to the deterministic solution where $\mathbf{S}^{(i)}\rightarrow 0$. As $\lambda \rightarrow 0$, there will be no shared response between individuals and all portions are unique to each individual. In other words $\mathbf{S}^{(i)}\rightarrow \mathbf{X}^{(i)}$.

**Number of Components** $\text{(n_components)}$

The n_components parameter sets the number of components we want our model to compute. This is algorithm is partially inspired by robust principal component analysis. 

### **Optimization Problem**
Equation (1) is then estimated by solving the following optimization problem

\begin{equation}\tag{2}
\min\limits_{\mathbf{S}^{(i)}, \mathbf{W}^{(i)}, \mathbf{R}}  \sum_{i=1}^{N} \frac{1}{2} ||\mathbf{X}^{(i)} - \mathbf{W}^{(i)}\mathbf{R} - \mathbf{S}^{(i)}||^2_F + \lambda_i||\mathbf{S}^{(i)}||_1\\
\text{s.t.}\ \mathbf{W}^{(i)^T}\mathbf{W}^{(i)} = \mathbf{I}, \ \ \forall i = 1 \dots N
\end{equation}

Equation (2) is a non-convex optimization problem, but we can use a greedy approach to estimate subsets of the model and combine the results at the end. Using Block Coordinate Descent, we can partition the variables into blocks and optimize each block while fixing the other blocks constant. In RSRM, each individual mapping from the latent space $\mathbf{W}^{(i)}$, each individual non-shared/unique matrix $\mathbf{S}^{(i)}$, and the shared response model $\mathbf{R}$ is a block. Because optimizing each of these blocks while keeping the other blocks constant is a convex problem, we can approximate the global optimum with a greedy solution. 

### **Algorithm's Process**
The derivations of $\mathbf{R}$, $\mathbf{W}^{(i)}$, and $\mathbf{S}^{(i)}$, will be provided above the functions in which they are used. Here we will discuss the broader process of the algorithm. 

The algorithms main process iterates these three steps. 

**(1) Solve for $\mathbf{W}^{(i)}$ with Procrustes:** 
<br />
\begin{equation}\tag{3}
\mathbf{W}^{(i)} = \mathbf{U}^{(i)}\mathbf{V}^{(i)^T}
\end{equation}
<br />
Where, $\mathbf{U}^{(i)}\mathbf{V}^{(i)^T}$ is achieved through the singular value decomposition of: 
<br />
<br />
\begin{equation}\tag{4}
\mathbf{U}^{(i)}\mathbf{\Sigma}^{(i)}\mathbf{V}^{(i)} = ( (\mathbf{X}^{(i)} - \mathbf{S}^{(i)} ) \mathbf{R}^T )
\end{equation}
<br />
<br />

**(2) Solve for $\mathbf{S}^{(i)}$ with Soft Shrinkage:**
<br />
\begin{equation}\tag{5}
\mathbf{S}^{(i)}=\text{Shrink(}\mathbf{X}^{(i)}-\mathbf{W}^{(i)}\mathbf{R},   \lambda)
\end{equation}
<br />
Where the amount of shrinkage is determined by $\lambda$. 
<br />
<br />
<br />
<br />
**(3) Solve for $\mathbf{R}$:**
<br />
\begin{equation}\tag{6}
\mathbf{R} = \frac{1}{N} \sum_{i=1}^{N}
\mathbf{W}^{(i)^T}(\mathbf{X}^{(i)}-\mathbf{S}^{(i)})
\end{equation}
<br />
<br />
Generally, the optimum can be achieved with a relatively low number of iterations which is ideal since we want to be able to update the shared response quickly if we gather more data.
