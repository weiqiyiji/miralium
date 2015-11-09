[Margin Infused Relaxed Algorithm](http://en.wikipedia.org/wiki/Margin_Infused_Relaxed_Algorithm) (MIRA), also called passive-aggressive algorithm (PA-I), is an extension of the perceptron algorithm for online machine learning that ensures that each update of the model parameters yields at least a margin of one.

For details, see: Crammer, K., Singer, Y. (2003): [Ultraconservative Online Algorithms for Multiclass Problems](http://jmlr.csail.mit.edu/papers/volume3/crammer03a/crammer03a.pdf). In: Journal of Machine Learning Research 3, 951-991.

Our java implementation follows [CRF++](http://crfpp.sourceforge.net/) file formats and can be used for [part-of-speech tagging](PosTaggerTutorial.md), [chunking](ChunkerTutorial.md) and other tagging tasks. Compared to CRFs, mira is much faster to train at the cost of a little reduction in accuracy, which is very valuable for sequence classification tasks with many labels.

NOTE: Does not support valued features, n-best output and confidence scores yet.

**NEWS: miralium is on [MLComp](http://mlcomp.org/) in the sequence tagging domain. You can benchmark it against other algorithms.**

**Building**

This project requires the [GNU trove library](http://trove4j.sourceforge.net/), included in the svn.
You can run Apache ant in the current directory to compile sources and build a jar, or you can use the following instructions:

  * Compile:

```
javac -source 1.5 -target 1.5 -cp trove-2.1.0.jar edu/lium/mira/*.java                                                                                                 
```

  * Create jar:

```
jar xf trove-2.1.0.jar gnu                                                                                                                                             
jar cfm mira.jar manifest.txt edu/ LICENSE LICENSE.trove gnu/                                                                                                          
rm -rf gnu/                                                                                                                                                            
```

**Running**

  * Execute:

```
java -Xmx2G -jar mira.jar                                                                                                                                              
```

The "mira" shell script is provided for convenience.

  * Usage:

```
TRAIN: java -Xmx2G Mira -t [-f <cutoff>|-s <sigma>|-n <iter>|-i <model>|-r|-iob|-fi] <template> <train> <model> [heldout]                                              
    Use training data for learning model parameters.                                                                                                                   
    -t          yes we do training                                                                                                                                     
    -f <cutoff> ignores features that occur less than <cutoff> times in the training data                                                                              
    -s <sigma>  sets the maximum magnitude of weight updates. A smaller value leads to slower, less overfitted convergence. 0.01 is often good.
    -n <iter>   number of training iterations. 5-10 is usually good.                                                                                                   
    -i <model>  load weights from an already existing model (for adaptation).                                                                                          
    -iob        assume multi-word labels when computing F-scores (use B- prefix on the first word, I- prefix for inside, and O for outside).                           
    -r          randomize starting weights (you can merge multiple models with random inits)                                                                           
    -fi         use frequency as starting weight                                                                                                                       
    <template>  template file for generating features (same as CRF++)                                                                                                  
    <train>     training examples                                                                                                                                      
    <model>     output model name                                                                                                                                      
    [heldout]   testing examples for estimating performance (non mandatory)                                                                                            
                                                                                                                                                                       
PREDICT: java -Xmx2G Mira -p <model> [test]                                                                                                                            
    Use learned model parameters to predict labels on new data.                                                                                                        
    -p          yes we do predictions                                                                                                                                  
    <model>     model weights to use                                                                                                                                   
    [test]      testing example file (can also be fed through stdin)                                                                                                   

CONVERT: java -Xmx2G Mira -c [<model> <model.txt>|<model.txt> <model>]
    Convert a model to the text format that can be understood by CRF++. The
    .txt extension is used to guess model format. In addition, use "crf_learn -C
    model.txt model.crf" to create a crf_test compatible model.

MERGE: java -Xmx2G Mira -m <output_model> <model1> <model2> ...
    Merge a few models by adding their weights. They must have been trained with
    the same template and same features (use that with random inital weights).
```

**Format**

The input format is the same as in the CRF++ toolkit
(http://crfpp.sourceforge.net/).  An instance corresponds to a sequence of
labels, one line per label. Instances are separated by blank lines. Each line
has a number of feature columns and the last element is the label itself. The
template file helps design combination of features that appear on consecutive
lines.

```
feature1_1 feature1_2 feature1_3 label1 (example1)
feature2_1 feature2_2 feature2_3 label2
feature3_1 feature3_2 feature3_3 label3
                                        <blank line>
feature4_1 feature4_2 feature4_3 label4 (example2)
feature5_1 feature5_2 feature5_3 label5
                                        <blank line>
```

**Templates**

A template mechanism helps generating n-gram features from the feature
columns.  The template file contains multiple lines, each line corresponding to
one composite feature.  If it starts with "U", then it's used to predict the
current label (and generates n weights given n different labels). If it starts with "B",
it's used to predict the label bigram (current, previous) and it generates n\*n
weights in the model. In a composite feature definition, `%x[i,j]` is replaced
with the value at line i and column j relative to the current label. In
our previous example, if we are processing label2, `%x[0,0]` is feature2\_1,
`%x[-1,1]` is feature1\_2 and `%x[1,3]` is feature3\_3. Example template:

```
U01%x[0,0]
U02%x[0,0]/%x[0,1]
U03%x[0,0]/%x[1,0]
B
```

Here, we use ids so that features are treated as separate weights in the model
in case they correspond to the same text. B means use label bigrams irrespective
of the features.

Make sure to check [the](ChunkerTutorial.md) [tutorials](PosTaggerTutorial.md) to know how to squeeze the most out of miralium.