---
title: Federated Learning in 15 minutes
author: ch4174nya
date: 2021-11-29 14:10:00 +0800
tag: federated learning
---
A similar version of this article was also published on [Medium](https://medium.com/@chaitanya_/federated-learning-in-15-minutes-b4b84562838b) in collaboration with colleagues. 

<!--more-->

The IBM Federated Learning (FL) library is a Python-based federated learning framework that empowers users to try their hand at federated learning with a number of popular models and fusion algorithms and different datasets. The scenarios that motivate FL, the components required in the FL setting as well as the challenges were detailed in a [prior article](https://research.ibm.com/blog/what-is-federated-learning).

In this article, we look at the IBM FL Experiment Manager, that I was led the development of. The Experiment Manager facilitates designing, setting up, launching, monitoring, and evaluating FL experiments using the IBM FL library and makes it as easy as training any standard machine learning task. It allows a user to get started and get results within an hour, even with no prior experience using federated learning.

## What is federated learning?
Consider the following scenario: a scientist at a hospital wishes to develop a model to predict which existing medications should be provided to patients to mitigate the symptoms of a disease. While some data exists in the hospital database, the number of patients presenting the disease and having taken different medication combinations remains too small to train an accurate machine learning model. Databases all over the globe with similar data exist; but they are guarded by laws or practices, preventing the data from being shared.

| <img src="{{site.url}}{{site.baseurl}}/assets/img/fl-network.webp" alt="Figure 1 from PETS paper" width="1000"/> | 
|:--:| 
| *Federated Learning use-case where multiple hospitals benefit from each others’ model parameters* |

Federated Learning (FL) is a way to apply machine learning to the data, allowing it to remain in place and thereby abiding by laws and regulations preventing data sharing. Thus by training the model through federated learning, scientists from across hospitals and even across country borders can leverage each others’ datasets to improve the training of their own models, obtaining far more accurate and wide-reaching models than they would have otherwise been able to, without ever seeing or touching the other hospitals’ datasets.

The Federated Learning paradigm allows training machine learning models without moving or sharing data by enabling participating entities to each train a local model and share only anonymous model parameters, never data. The parties’ model parameters are exchanged with a central aggregator using the FL protocol. The aggregator fuses (aggregates) the results from the different parties and communicates the consolidated model parameters back to the parties, who use the parameters in their local training on their own data, and the process repeats. The benefit lies in the model parameters having been aggregated across models trained locally on many different datasets. The FL training process continues in this manner for multiple rounds until a stopping criterion is met.

## The IBM FL Experiment Manager
To bring the potential of FL to a wider public, researchers and engineers at IBM Research developed the IBM Federated Learning [library] (https://github.com/IBM/federated-learning-lib). The IBM FL library facilitates configuring and executing FL training in an enterprise setting.

As discussed in this post, the IBM FL library supports deep neural networks, linear regression, and k-means, along with many other supervised and unsupervised machine learning and statistical methods. Moreover, the library provides users the ability to add new fusion algorithms, specific models, and so on.

The IBM FL Experiment Manager is a new feature added to the IBM FL library. It allows users to easily design, set up, deploy and monitor their experiments locally, on remote virtual machines, and in multi-cloud settings using RedHat OpenShift, all through a very easy-to-use Jupyter Notebook dashboard.

## Why use the Experiment Manager?
The IBM FL library offers users a powerful command-line interface that gives a great deal of freedom to experienced developers. The power of a command-line interface comes with added responsibilities: users working with command-line interfaces must manually track multiple windows, the protocol steps, and the postprocessing of results at the end of the run.

The IBM FL Experiment Manager was designed to eliminate this burden and give users, especially new users, a fast jumpstart into FL. It does everything including
    - helping users choose the right parameters and hyperparameters,
    - setting up, running, and monitoring the aggregator and party processes (or containers, depending on the choice of deployment), and
    - visualizing the results in terms of loss and accuracy plots at the end of the FL run

The IBM FL Experiment Manager makes it easy for users to orchestrate and manage the FL experiments and even a very inexperienced user can be up-and-running in minutes.

## Getting Started

To try out the IBM FL Experiment Manager, head over to the [Github page of the IBM FL library](https://github.com/IBM/federated-learning-lib) and set it up on your machine using the instructions given in the Readme. Once the library is set up, navigate to the `experiment_manager` directory and follow the instructions in the `usage_guide.md` file.

The cells of the Experiment Manager allow the user to choose the model (Keras, PyTorch, scikit learn, TensorFlow), a supported fusion algorithm, a compatible dataset (there’s also support for bringing your own dataset), and how the data should be split across parties.

Users can choose the number of parties to participate in the federation, as well as edit the hyperparameters for training. These include the number of rounds, the number of epochs in each round as well as the learning rate and stopping threshold for training, all of which will depend on the model and fusion algorithm chosen.

Lastly, a notebook cell takes in the user’s desired deployment parameters, drawing upon the mode chosen — whether running locally, on remote virtual machines, or on OpenShift Cloud Platform. These inputs are provided through Jupyter Notebook widgets rendered within the notebook cells.

<iframe src="https://player.vimeo.com/video/646811481?badge=0&amp;autopause=0&amp;player_id=0&amp;app_id=58479" width="720" height="405" frameborder="0" allow="autoplay; fullscreen; picture-in-picture; clipboard-write" title="IBM FL Experiment Manager"></iframe>

Once these choices have been made, the Experiment Manager displays the configuration settings for the aggregator and parties. After that, the user can easily launch the experiment. monitor and evaluate it. As the experiment progresses, completion bars indicate near real-time progress in terms of the rounds and party responses in each round. Once the experiment is finished, visualizations in the form of loss and accuracy plots are displayed. This flow is shown in the short video above.


## Summary
The Experiment Manager is packaged within the IBM FL library. It provides a single window to carry out an entire FL experiment, from choosing federation parameters to postprocessing, including the choice of training-related hyperparameters. This allows a user, especially a new FL user, to get started in minutes on trying one’s hand at federated learning. The IBM FL Experiment Manager eliminates the need to use a terminal, monitor multiple windows simultaneously, and comb through different directories and files to examine loss and accuracy results, providing all of these core FL tasks in an easy-to-use Jupyter Notebook.

IBM FL Experiment Manager makes it easier than ever to get started and to focus on the FL components of interest — the data, the algorithms, and the role of each party in the federation — without having to worry about the runtime logistics.

---
*Disclaimer: The views expressed in the article are my own and do not reflect the position of any of the entities mentioned in the article.*
