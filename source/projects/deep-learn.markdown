---
layout: page
title: "Deep Learning for Image Classification"
date: December 15, 2014 
comments: true
sharing: true
footer: true
---


Finding good features is an important yet challenging task in most machine learning applications. For computer vision tasks, deep learning techniques, based on multi-layer neural networks, are effective in extracting good learning representations from image data. Although powerful, deep neural networks are prone to overfitting. Therefore, we study different regularization techniques that mitigate this overfitting. In the following, we focus on the feed-forward neural network and describe its mechanisms and regularization techniques. Finally, we describe an implementation of neural network using the new Julia programming language with an example experiment of hand-written digit classification.


## Deep Neural Networks 

Neural networks are biologically inspired machine learning models that offer lots of flexibility in modeling a target function. It is composed of multiple layers of neurons and was shown to be a universal approximation [1]. Although flexible, neural network can easily overfit. Therefore, regularization techniques are needed to train neural networks. In this project, we have investigated different regularization techniques. The objective of the project is to implement a prototype neural network library in Julia,which can be used to investigate different regularization techniques and optimization methods. Note that this work is still in progress. Here, we present a thorough review of neural networks and describe different regularization techniques. Finally, we describe the implementation before giving our concluding remarks.

### Feed-forward Neural Networks
In this section, we present the basic model of feed-forward neural networks. Next, we present the procedures for fitting the network to the training data. First, the backpropagation algorithm, which is used to compute the network weights, is derived for the case of multi-label classification. Next, we will show how different optimization procedures can be built on top of the backpropagation algorithm to minimize the cross-entropy training error.


### Multi-layer Network Model

Let’s start by describing the architecture of the neural network. As illustrated in Figure , it consists of L layers of connected neurons or units. The layers are index as l = 1, . . . ,L, in which the first L − 1 layers are hidden and the last layer l = L is the output layer. The input data x = [1, x1, . . . ,xp], which has p raw features, enters the network from the leftmost units. It then flows through the network towards the output nodes on the right. Each layer l has d(l) neurons and a bias node. The bias node outputs a constant value 1, which corresponds to an intercept term. On the other hand, a neuron transforms the input, also called activation, to an output using a nonlinear operation ✓. The resulting output is called the activity of the neuron. Except for the bias node, all nodes between two consecutive layers are connected by arrows. In particular, the arrow between node i of layer l and node j of the next layer multiplies the activity x(l)
i of layer l by the weight w(l)ij and passes the result to node j. All activities of layer l that goesinto node j is combined to obtain the activation s(l)j as follows:

{% img right /images/net.png 250 250 %}

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


### Model fitting using backpropagation

Now, we consider the question of how to set up the neural network. Since it is computationally hard to find the optimal size and depth of the network, a good choice is conventionally found by trial-anderror. However, in Section (3), we will present an efficient technique called dropout to generate and average the predictions of many network configurations. In this section, we will focus on finding the optimal weight parameter w given a configuration and training data.

The next thing we need is an error measure for the multi-label classification task. Assuming the class conditional distribution of the training data is multinomial, we employ the negative likelihood as the error function: 

where xn is a feature input vector and yn is a C⇥1 vector that encodes the class label using one-of-C rule. The function h is parameterized by the weights w. In (4), we can identify E as a cross-entropy error function that measures the “distance” between the estimated MAP probabilities h (xn;w) and the true label vector yn.

The error surface E is not convex w.r.t. the weights w. Nonetheless, its derivatives are still useful for finding directions towards a good local optimum. In fact, finding the global optimum is not only computationally prohibitive, but also it can be harmful. In Section (3), we will discuss the importance to early stopping our search to avoid overfitting. In any case, we always need a way to cheaply compute all partial derivatives. Fortunately, there is an efficient algorithm, called the backpropagation [3] that has O (M) run-time complexity. Here,M is the total number of weights in the network. 

## Model fitting and prediction algorithms

Previously, we have described the implementation of deep neural network models. e describe the main functions of the neural network implemented with Julia programming
language. 


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
@debug function fit!(nn::NeuralNet, X, y, options::FitOptions, maxIter::Int)

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



#### Forward propagation 

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
@debug function forwardPropagate(nn::NeuralNet, xn::Array{Float64})
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

#### Backward propagation

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

## Optimization methods

Next, let’s present different iterative methods for optimizing the weights w(l)ij . There are two general strategies for doing this:

1. Batch gradient descent update the weights using the gradient of the error contributed by all training examples. This is done after a single pass through all the examples as follows:
2. Stochastic gradient descent (SGD) is a more efficient method as it immediately updates the
weights after seeing each training example[4]. Another advantage of the stochastic approach is that it takes advantage of
randomization to escape poor local optimum. The overall SGD algorithm is presented in
Table 2.

Different acceleration techniques have also been proposed to speed up the training procedure of neural networks. These include Nesterov’s method, conjugate gradient descent (which only works for batch mode) as well as Newton-type methods. Due to the computational hurdles with training large neural networks, first-order methods such as SGD are more practical.

## Regularization techniques
Neural networks offer a considerable flexibility to the extent that they can approximate any function with abritrary complexity. Therefore, they can also easily overfit the data and fail to generalize to give accurate predictions when presented with new data samples. To prevent this, we can use three types of regularization methods: weight decay, early stopping and dropout.

#### Weight decay
Weight decay is a classic regularization technique [5] that is used in different problems, including linear ridge regression. The basic idea is to penalize models that are too complex by adding a quadratic penalty terms to the error loss function and the weight update becomes:
Precisely, it seeks to shrink the weights towards zero.

#### Early stopping
The second regularization method avoids overfitting using a validation set approach. Precisely, we monitor the prediction error of the validation set at each iteration of the weight optimization. Although the optimization seeks to minimize the training error, we stop as early as the validation error increases with the iterations. This is a good indication that the model starts to overfit.

#### Dropout
Dropout is another regularization technique that was recently proposed in [6]. The key idea is to randomly drop the units during training with a probability p. When a unit is dropped, all its incoming and outgoing connections are also temporarily removed. For each training point, we sample the network by droping units randomly to produce a “thinned” network. This is usually done within a mini-batch and the weight of each arrow becomes average over the points in the mini-batch. If for one point, the unit connecting the arrow was dropped, its weight counts as zero. At test time, the entire network is used but the weights are scaled by the probability p.	


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
```


#### Weight initialization

```julia
##----------------------------------------------------------------------
## initWeights: initializate weights matrices
## Inputs:
##		- Neural network structure
##		- Sensitivity matrices Δ[l], ∀l ∈ {1,..,L}
##		- Output vectors x[l], ∀l ∈ {1,..,L+1} 
##		- Class label of example: yn 
##----------------------------------------------------------------------

function initWeights!(nn::NeuralNet, X::Array{Float64})
```


## References 

1. K Murphy. Machine Learning: A Probabilistic Perspective. The MIT Press, 2012.
2. C Bishop. Neural Networks for Pattern Recognition. Oxford University Press, Inc., New York, NY, USA, 1995.
3. D Rumelhart, G E Hinton, and R J Williams. Learning representations by back-propagating errors. Nature, 323(6088):533–536, 1986.
4. L Bottou. Stochastic gradient learning in neural networks. In In Proceedings of Neuro-Nimes. EC2, 1991.
5. A Tikhonov. Solution of incorrectly formulated problems and the regularization method. Soviet Math. Dokl., 4:1035–1038, 1963.
6. N Srivastava, Geoffrey Hinton, Alex Krizhevsky, Ilya Sutskever, and Ruslan Salakhutdinov. Dropout: A simple way to prevent neural networks from overfitting. Journal of Machine Learning Research, 15:1929–1958, 2014.
7. M Schmidt, N Le Roux, and F Bach. Minimizing finite sums with the stochastic average gradient. CoRR, abs/1309.2388, 2013.
