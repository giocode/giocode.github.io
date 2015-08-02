---
layout: page
title: "Deep Learning for Image Classification"
date: December 15, 2014 
comments: true
sharing: true
footer: true
---


Finding good features is an important yet challenging task in most machine learning applications. For computer vision tasks, deep learning techniques, based on multi-layer neural networks, are effective in extracting good learning representations from image data. Although powerful, deep neural networks are prone to overfitting. In this project, I studied different regularization techniques that mitigate  overfitting. Precisely, we present the basic model of feed-forward neural networks. Next, we present the procedures for fitting the network to the training data. In particular, the backpropagation algorithm, which is used to compute the network weights, is derived for the case of multi-label classification. Next, we will show how different optimization procedures can be built on top of the backpropagation algorithm to minimize the training error. In addition, a prototype neural network library is implemented in the programming language Julia and described in the following.

## Deep Neural Networks 

Neural networks are biologically inspired machine learning models that offer lots of flexibility in modeling a target function. It is composed of multiple layers of neurons and was shown to be a universal approximation [1]. Although flexible, neural network can easily overfit. Therefore, regularization techniques are needed to train neural networks. The focus of the project is to investigate different regularization techniques. 

### Feed-forward Neural Network Model


Let’s start by describing the architecture of the neural network.  As illustrated in the following figure, it consists of \\(L\\) layers of connected neurons or units. The layers are index as \\(l = 1, . . . ,L\\), in which the first \\(L − 1\\) layers are hidden and the last layer \\(l = L\\) is the output layer. 

{% img ../images/neural/net.png %}

The input data \\(x = [1, x_1, . . . ,x_p]\\), which has p raw features, enters the network from the leftmost units. It then flows through the network towards the output nodes on the right. Each layer \\(l\\) has \\(d^{(l)}\\) neurons and a bias node. The bias node outputs a constant value 1, which corresponds to an intercept term. On the other hand, a neuron transforms the input, also called activation, to an output using a nonlinear operation \\(\theta\\). The resulting output is called the activity of the neuron. Except for the bias node, all nodes between two consecutive layers are connected by arrows. In particular, the arrow between node \\(i\\) of layer \\(l\\) and node \\(j\\) of the next layer multiplies the activity \\(x^{(l)}\\)
i of layer l by the weight w(l)ij and passes the result to node \\(j\\). All activities of layer \\(l\\) that goes into node \\(j\\) is combined to obtain the activation \\(s^{(l)}_j\\) as follows:

$$s^{(l)}_j = $$

This later is then transformed to an activation that is passed to all nodes of the next layer:

$$x^{(l+1)}_j = $$

By concatenating all activation inputs and outputs (including the bias nodes’), the above operations can be concisely described in vector notations:

$$s^{(l)} =$$

$$x^{(l+1)} =$$

For classification tasks, the dimension \\(d^{(L)}\\) of the network output vector \\(x^{(L+1)}\\) is equal to the number of class labels \\(C\\). In fact, \\(x^{(L+1)}\\) models the posterior probabilities that a sample belongs to each class given the feature data x^{(1)}. To enforce that the outputs sum to one, we use the softmax activation function:

$$x^{(L+1)}_j$$

For the hidden layers, we instead use the hyperbolic tangent function:

$$x^{(l+1)}_j = $$

Unlike x(L+1), the output vectors x(l) of the hidden layers all include a bias \\(x^{(l)}_1 = 1\\). Thus, \\(x^{(l)}\\) has \\(d^{(l)} + 1\\) elements for all \\(l = 1, . . . ,L\\). These moving parts of a neural network are summarized in the Table 1 and the chain of transformations from input to output is illustrated below:

In summary, a neural network is characterized by: 
- the total number of layers 
- the input and output dimensions 
- the number of units at each __hidden layer__ (i.e. layer between input and output)
- an activation function at each neuron or node
- a set of weights matrices that relate the input and output between layers.

In Julia, we can define a neural network by the following data type:

```julia
##----------------------------------------------------------------------
## Neural Network structure 
type NeuralNet
	numLayers::Int   					# Number of layers (excluding input)		
	numInputs::Int 						# Input dimension (no bias node)
	numOutputs::Int 					# Output dimension
	numHidden::Array{Int} 				# Number of units at each hidden layer
	activFunc::String 					# Activation function of hidden units
	weights 						   	# Weight matrices  
end
```

#### Predicting the class of test samples

{% img ../images/neural/forward.png %}

Given these parameters, the relationship between the inputs and the outputs of the network is determined by a process called _**forward propagation**_. When predicting the class label of a new test sample \\(\mathbf{x} \\) the corresponding outputs \\(h_k(\mathbf{x}) = x^{(L+1)}_k\\)  \\(k = 1, . . . C\\) are calculated using the chain of transformations previously described. Implementing the prediction is very simple. All we need to do is to run  the forward propagation algorithm with the new sample `xnew`:


```julia
##----------------------------------------------------------------------
## predict: predict the labels of new samples using forward propagation 
## Inputs:
##		- Neural network structure
##		- New samples of size N x p 
## Output: a vector of pairs containing:
##		- Maximum posterior probability of predicted class 
##		- Label of predicted class ∈ {1,...,C}
##----------------------------------------------------------------------
function predict(net::NeuralNet, xnew)
	x,s = forwardPropagate(net, xnew)
	prob, class = findmax(x[end])
end
```

The implementation of forward propagation follows directly from the above equations as follows: 

```julia
##----------------------------------------------------------------------
## Forward propagation subroutine
## Inputs: 
## 		- Training example: xn
## 		- Neural network structure: number of layers, number of units
## Outputs: 
##		- L+1 output vectors x[l] (including input vector x[1] = xn)
##		- L input vectors s[l] 
##----------------------------------------------------------------------
function forwardPropagate(nn::NeuralNet, xn::Array{Float64})
	# println("Forward propagation..")
	# Network topology
	L = nn.numLayers   					 # number of all layers excluding input 
	C = nn.numOutputs

	# Cells containg L input vectors and L+1 output vectors
	s = cell(L,1) 	# inputs of all layers 
	x = cell(L+1,1) # outputs of all layers

	# Compute input s[l] and output x[l+1] at each layer
	# Input vector x[1] has no bias node
	x[1] = xn 
	# Hidden layers
	for l in 1:L-1
	  	W = (nn.weights[l])
	  	s[l] = W'*x[l] 
	  	x[l+1] = [1, map(tanh, s[l])]   # use tanh
	end
	# Output layer
	s[L] = (nn.weights[L])' * x[L]
	x[L+1] = if C == 1 
             logit(s[L])
           else 
           	 softmax(s[L])
           end

	# return output and input vectors
    x, s
end
```


## Model fitting using backpropagation

Previously, we assumed that the neural network model is already trained, i.e. the set of weight matrices are pre-configured. But how do we set these weight matrices \\(W^{(l)}, l=1,...,L\\)? In the following, we answer that question using another procedure called __**backpropagation**__ in conjunction with forward propagation. A related question is how big and how deep should the network be. Since, it is computationally prohibitive to find the optimal size and depth of the network, a good choice is conventionally found by trial-and-error. However, we will later present an efficient technique called _**dropout**_ to generate and average the predictions of many network configurations. In this section, we will focus on finding the optimal weight parameter \(w given a configuration and training data.

The next thing we need is an error measure for the multi-label classification task. Assuming the class conditional distribution of the training data is multinomial, we employ the negative likelihood as the error function: 

where \\(x_n\\) is a feature input vector and \\(y_n\\) is a \\(C\\) by  \\(1\\) vector that encodes the class label using _one-of-C rule_. The function \\(h\\) is parameterized by the weights \\(w)\\. In (4), we can identify \\(E\\) as a cross-entropy error function that measures the “distance” between the estimated MAP probabilities \\(h(xn;w)\\) and the true label vector \\(yn\\).

The error surface \\(E\\) is not convex w.r.t. the weights \\(w\\). Nonetheless, its derivatives are still useful for finding directions towards a good local optimum. In fact, finding the global optimum is not only computationally prohibitive here, but also it can be harmful. We will discuss more about this issue later and emphasize the importance to stopping our search  early to avoid overfitting. In any case, we always need a way to cheaply compute all partial derivatives. Fortunately, there is a clever algorithm, called the _backpropagation_ [3] that does it with \\(O(M)\\) runtime complexity. Here, \\(M\\) is the total number of weights in the network. 

The implementation of the model training or fitting function is shown below: 

```julia
##----------------------------------------------------------------------
## Fit: train the neural network model 
## Inputs:
##		- Neural network structure
##		- Matrix of features X with dimension N x p
##		- Class label vector of dimension N x 1 (yn ∈ {1,...,C})
##		- fitting options 
##		- maximum number of optimization iterations 
##----------------------------------------------------------------------
function fit!(nn::NeuralNet, X, y, options::FitOptions, maxIter::Int)

	# assume trainData if preprocessed to make learning faster
	# e.g. successive examples come from different classes 
	L = nn.numLayers
	C = nn.numOutputs
	N = size(X,1)
	d = [nn.numInputs, reshape(nn.numHidden, length(nn.numHidden),1), nn.numOutputs]

	# Initialize weights 
	initWeights!(nn, X)

	# Iteration of whole batch training
	for i = 1:maxIter
		if i % 10 == 0 tic; println("Pass data Iteration: $i") end
  	for n = 1:N 
  		if n % 50 == 0 println("Samples: $n") end
  		# Local variables within scope of each iteration
  		xn, yn = X[n,:]', y[n]
 		
  		# Compute output vectors x(l), l = 1,...,L+1
  		x, s = forwardPropagate(nn, xn)

  		# Compute sensitivity matrices Δ[l], l = 1,...,L	 
  		local Δ = backPropagate(nn, x)

  		# Update the weights
  		updateWeights!(nn, Δ, x, yn)
  	end  
  	if i % 10 == 0 toc end  	
	end

  # return in-sample error (optionally)
end
```

Basically, the training is performed after multiple iterations of both the forward and backward propagations. Through these rounds, the neural network is trained to model the right relationship between the input and the output. The backward propagation computes the sensisitivy matrices \\(\Delta\\) as follows: 

```julia
##----------------------------------------------------------------------
## Backpropagation subroutine
## Inputs: 
##		- L+1 output vectors x[l] (including input vector x[1] = xn)
##		- L input vectors s[l] 
## Outputs: 
## 		- Sensitivity matrices Δ[l] for all layer l = 1,...,L
##----------------------------------------------------------------------
function backPropagate(nn::NeuralNet, x)
	# println("Back propagation..")
	# Network topology
	L = nn.numLayers   					 # number of all layers excluding input 
	C = nn.numOutputs
	d = [nn.numInputs, reshape(nn.numHidden, length(nn.numHidden),1), nn.numOutputs]
	Δ = cell(L,1) 	

	# Output vector
	h = x[L+1]	

	# Sensitivity matrix for output layer
	Δ[L] = [ h[i]*(δ(i,j) - h[j]) for i in 1:C, j in 1:C ]

	# Sensitivity matrix for inner layers
	for l = L-1:-1:1
		Δ[l] = zeros(Float64, C, d[l+1]) 
		W = nn.weights[l+1]
		for k in 1:C
			for j in 1:d[l+1]
				# assuming tanh activation function at hidden units				
				Δ[l][k,j] = (1 - h[k]^2) * (W[j,:] * Δ[l+1][k,:]')[1]
			end
		end
	end

	# return sensitivity matrices for all layers ∀l = L,...,1
	Δ
end
```

## Optimization the weights

After the sensitivity matrices are calculated by the weight matrices, the weight matrices are optimized. There are two general strategies for doing this:

1. Batch gradient descent update the weights using the gradient of the error contributed by all training examples. This is done after a single pass through all the examples.
2. Stochastic gradient descent (SGD) is a more efficient method as it immediately updates the weights after seeing each training example[4]. Another advantage of the stochastic approach is that it takes advantage of randomization to escape poor local optimum. The overall SGD algorithm is presented in Table 2.

{% img right ../images/neural/table2.png %}

Different acceleration techniques have also been proposed to speed up the training procedure of neural networks. These include Nesterov’s method, conjugate gradient descent (which only works for batch mode) as well as Newton-type methods. Due to the computational hurdles with training large neural networks, first-order methods such as SGD are more practical. The implementation of simple SGD method is displayed below:


```julia
##----------------------------------------------------------------------
## updateWeights: update and return the weight matrices at each iteration
## Inputs:
##		- Neural network structure
##		- Sensitivity matrices Δ[l], ∀l ∈ {1,..,L}
##		- Output vectors x[l], ∀l ∈ {1,..,L+1} 
##		- Class label of example: yn 
## Output: 
##		- weights matrices
##----------------------------------------------------------------------
function updateWeights!(nn::NeuralNet, Δ, x, yn)

	L = nn.numLayers
	C = nn.numOutputs
	d = [nn.numInputs, reshape(nn.numHidden, length(nn.numHidden),1), nn.numOutputs]

	# Softmax output
	h = x[L+1]

	# Step size
	μ = 0.1

	# yn is encoded as one-of-C vector
	yv = mask(ones(C), yn)

	# Gradient of error w.r.t softmax output vector
	∇e_h = - vec(yv ./ h)

	# Gradient of error w.r.t weight matrix W[L] 
	# assuming softmax output activation and cross-entropy error
	∇e_WL = zeros(Float64, d[L]+1,C)
	for i in 1:d[L]+1
	  for j in 1:C 
			∇e_WL[i,j] = x[L][i] * (h[j] - yv[j]) 
		end
	end
	# SGD update of weight matrix W[L]
	nn.weights[L] = nn.weights[L] - μ * ∇e_WL

	# Gradient of error w.r.t weight matrix @ input W[1] 
	# Input has no bias node: W1 is d[1] x d[2] matrix
	∇e_W1 = zeros(Float64, d[1],d[2])
	for i in 1:d[1]
	  for j in 1:C 
			∇e_W1[i,j] = x[1][i] * dot(∇e_h, vec(Δ[1][:,j]))
		end
	end

	# SGD update of weight matrix W[1]
	nn.weights[1] = nn.weights[1] - μ * ∇e_W1

	# Gradient of error w.r.t weight matrices of hidden layers
	for l = 2:L-1  
		# Gradient of error w.r.t weight matrix Wl
		# x[l] has bias node: Wl is (d[l]+1) x d[l+1] matrix
		∇e_Wl = zeros(Float64, d[l]+1, d[l+1])
		for i in 1:d[l]+1
			for j in 1:d[l+1] 
				∇e_Wl[i,j] = x[l][i] * dot(∇e_h, vec(Δ[l][:,j]))
			end
		end

		# SGD update the weight matrices 
		nn.weights[l] = nn.weights[l] - μ * ∇e_Wl
	end
	nn.weights
end	
```

## Regularization techniques
Neural networks offer a considerable flexibility to the extent that they can approximate any function with abritrary complexity. However, they can also easily overfit the data and fail to generalize to give accurate predictions when presented with new data samples. To prevent this, we can use three types of regularization methods: weight decay, early stopping and dropout.

#### Weight decay
Weight decay is a classic regularization technique [5] that is used in different problems, including linear ridge regression. The basic idea is to penalize models that are too complex by adding a quadratic penalty terms to the error loss function and the weight update becomes:
Precisely, it seeks to shrink the weights towards zero.

#### Early stopping
The second regularization method avoids overfitting using a validation set approach. Precisely, we monitor the prediction error of the validation set at each iteration of the weight optimization. Although the optimization seeks to minimize the training error, we stop as early as the validation error increases with the iterations. This is a good indication that the model starts to overfit.

#### Dropout
Dropout is another regularization technique that was recently proposed in [6]. The key idea is to randomly drop the units during training with a probability p. When a unit is dropped, all its incoming and outgoing connections are also temporarily removed. For each training point, we sample the network by droping units randomly to produce a “thinned” network. This is usually done within a mini-batch and the weight of each arrow becomes average over the points in the mini-batch. If for one point, the unit connecting the arrow was dropped, its weight counts as zero. At test time, the entire network is used but the weights are scaled by the probability p.	



## References 

1. K Murphy. Machine Learning: A Probabilistic Perspective. The MIT Press, 2012.
2. C Bishop. Neural Networks for Pattern Recognition. Oxford University Press, Inc., New York, NY, USA, 1995.
3. D Rumelhart, G E Hinton, and R J Williams. Learning representations by back-propagating errors. Nature, 323(6088):533–536, 1986.
4. L Bottou. Stochastic gradient learning in neural networks. In In Proceedings of Neuro-Nimes. EC2, 1991.
5. A Tikhonov. Solution of incorrectly formulated problems and the regularization method. Soviet Math. Dokl., 4:1035–1038, 1963.
6. N Srivastava, Geoffrey Hinton, Alex Krizhevsky, Ilya Sutskever, and Ruslan Salakhutdinov. Dropout: A simple way to prevent neural networks from overfitting. Journal of Machine Learning Research, 15:1929–1958, 2014.
7. M Schmidt, N Le Roux, and F Bach. Minimizing finite sums with the stochastic average gradient. CoRR, abs/1309.2388, 2013.
