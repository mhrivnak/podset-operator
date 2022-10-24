# podset-operator

Welcome to the PodSet operator exercise!

In order to learn fundamentals of operator development, your goal is to follow
the included tutorial to create an operator for a custom resource called
PodSet.

## Features

The PodSet resource is similar to the Kubernetes ReplicaSet, but without
so many features to worry about.

Users will create a PodSet resource and specify a size, denoting how many pods
they want running in their set of pods. The controller you write will need to
perform the following each time the Reconcile function runs:

1. figure out how many matching pods already exist
1. compare that to the desired number of pods, as specified in PodSet.Spec
1. add or remove pods as needed

For the purposes of this exercise, it doesn't matter what container image runs
in the pods.

## Tutorial

[Access the tutorial](./tut.md) to get started. As needed, you can reference the
Go code in this repository, but you are encouraged to start fresh and follow the
tutorial step-by-step to build your own.
