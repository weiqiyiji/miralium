# Introduction #

Part-of-speech tagging consists in annotating words with labels that reflect their syntactic role. For instance, the words of the sentence "It is raining" would be annotated as "It/PRP is/VBZ raining/VBN" where PRP stands for personal pronoun, VBZ stands for verb (present, 3rd person), and VBN stands for past participle. See the [list](http://bulba.sdsu.edu/jeanette/thesis/PennTags.html) defining the word-level tags in the Penn Treebank. Part-of-speech tagging (also called POS tagging) is used in many NLP tasks that don't require a full constituency parse tree but some information about the categories of words.

In this tutorial, we show how to use the [CoNLL 2000](http://www.cnts.ua.ac.be/conll2000/chunking/) data to train a POS tagger for English.

First, create a directory from the mira directory:
```
mkdir pos_tagger
cd pos_tagger
```

# Downloading the data #

Download the data from the CoNLL website, uncompress it and get rid of the [chunking](ChunkerTutorial.md) labels.
```
wget http://www.cnts.ua.ac.be/conll2000/chunking/train.txt.gz -O - | gunzip | cut -f1,2 -d" " > pos.train.txt
wget http://www.cnts.ua.ac.be/conll2000/chunking/test.txt.gz -O - | gunzip | cut -f1,2 -d " " > pos.test.txt
```

Examine the data: you see one word and one label per line. Sentences are separated by blank lines.

```
Confidence NN
in IN
the DT
pound NN
is VBZ
widely RB
expected VBN
to TO
...
```

# Building a simple tagger #

There are two problems that we need to tackle to be able to predict part-of-speech tags. Some words can have different labels depending on their role in the sentence. For instance, "pay" is used as a verb (base form VB and present form VBP) and a as noun (NN). The second problem is unseen words for which we don't know in advance their tags. For instance number are given the CD tag. But there is an infinity of numbers which are not in the training data.

Our classifier will have to learn to deal with both problems. In HMM taggers, this is generally handled by estimating the probability for a word to have a given tag (to help with the ambiguity problem) and the probability for a tag to be followed by an other tag (to help with generalization on unseen words). We can do similarly by defining a feature for the current word and the associated tag, and a feature for the current tag given the previous tag. At training, mira determines the optimal weight for each of these features.

## Defining a template ##

In mira, feature generation is defined in a template file. This file contains a template specification on each line which is applied on each input word (or exactly each line in the input file) and generates features accordingly.

```
cat > pos-simple.template <<EOF
U%x[0,0]
B
EOF
```

In this template definition, if the line begins by `U`, it generates tag-unigram features which have one weight for each tag. If it begins by `B`, it generates tag-bigram features which have one weight for each pair of consecutive tags. When predicting the labeling of a word, we take the tag with the highest score (sum of the  weights of the features for that tag).

When using tag-bigram features, the decision on a given word depends of the decision on the previous word which in turn depends on the decision on the word before it. We use dynamic programming (called in this case Viterbi decoding) to find efficiently the tag assignment with the maximum global score.

To come back to our template definition, features are generated according to slots from the input. In a template, `%x[i,j]` is replaced with the token in the input at the relative line/column position compared to the current word. Since we have only one column (the words), `%x[0,0]` is the current word (`%x[-1,0]` would be the previous word or a place-holder at the beginning of a sentence). So `U%x[0,0]` generates one weight for each tag, given the current word. This feature is helpful for determining how to tag a known word. `B` generates one weight for each pair of tags regardless of the input. This feature is helpful for contextualizing the tagging of a known word and using the context only in the case of an unseen word (note that in training there are no unseen words, which might bias the distribution of the weights).

## Training the classifier ##

```
../mira -t -f 2 -s 0.01 -n 10 pos-simple.template pos.train.txt pos-simple.model pos.test.txt
read 2 templates from "pos-simple.template"
counting: 8936
unigrams: 19122, bigrams: 1
unigrams: 9674, bigrams: 1, cutoff: 2
labels: 44
model: 427592 weights
iteration 0
  train: 8936 examples, terr=0.1504 fscore=0.8496
  test: 2012 examples, terr=0.1103 fscore=0.8897 (ok=42149 ref=47377 hyp=47377)
iteration 1
  train: 8936 examples, terr=0.0772 fscore=0.9228
  test: 2012 examples, terr=0.0929 fscore=0.9071 (ok=42978 ref=47377 hyp=47377)
iteration 2
  train: 8936 examples, terr=0.0631 fscore=0.9369
  test: 2012 examples, terr=0.0914 fscore=0.9086 (ok=43046 ref=47377 hyp=47377)
iteration 3
  train: 8936 examples, terr=0.0568 fscore=0.9432
  test: 2012 examples, terr=0.0835 fscore=0.9165 (ok=43421 ref=47377 hyp=47377)
iteration 4
  train: 8936 examples, terr=0.0540 fscore=0.9460
  test: 2012 examples, terr=0.0788 fscore=0.9212 (ok=43645 ref=47377 hyp=47377)
iteration 5
  train: 8936 examples, terr=0.0523 fscore=0.9477
  test: 2012 examples, terr=0.0755 fscore=0.9245 (ok=43798 ref=47377 hyp=47377)
iteration 6
  train: 8936 examples, terr=0.0514 fscore=0.9486
  test: 2012 examples, terr=0.0755 fscore=0.9245 (ok=43798 ref=47377 hyp=47377)
iteration 7
  train: 8936 examples, terr=0.0501 fscore=0.9499
  test: 2012 examples, terr=0.0754 fscore=0.9246 (ok=43807 ref=47377 hyp=47377)
iteration 8
  train: 8936 examples, terr=0.0488 fscore=0.9512
  test: 2012 examples, terr=0.0730 fscore=0.9270 (ok=43920 ref=47377 hyp=47377)
iteration 9
  train: 8936 examples, terr=0.0489 fscore=0.9511
  test: 2012 examples, terr=0.0683 fscore=0.9317 (ok=44142 ref=47377 hyp=47377)
writing model: pos-simple.model
```

Here, we train a model using the templates with have define, over 10 iterations (`-n 10`). We also prune out features that occur only once (`-f 2`) because they tend to help the classifier identify those examples where they occur and therefore put too much confidence in their presence, resulting in an _overfitted_ model. We also limit the magnitude of the weight updates to a small factor (`-s 0.01`) to smooth the learning process. After 10 iterations, our model performs at an accuracy of 93% on the test set. Not bad.

# Enhancing the tagger #

Our tagger could clearly benefit from more context and better generalization properties. First, instead of conditioning the current tag only on the current word, we can extend that to neighboring words. Then, in order to better generalize on unseen (but also on seen) words, we can add morphological features. For instance, if a word contains digits, it's likely to be a cardinal number; if it is capitalized, it's likely to be a proper name; if it finishes in "s" or "ed", it's likely to be a verb, and so on.

## Adding morphological features ##

In order to add morphological features, you can use the script `../examples/pos_features.py` which conveniently generates them for you. This scripts adds eleven features:
  * contains number (y/n)
  * is capitalized (y/n)
  * contains symbol (y/n)
  * prefixes from one to four letters (4 features)
  * suffixes from one to four letters (4 features)
Note that for the prefixes and suffixes, if a word is shorter than the number of expected letters, a special symbol ('nil') is used instead of the letters.

Using the data used to train the simple tagger, you can run:

```
python ../examples/pos_features.py < pos.train.txt > pos-enhanced.train.txt
python ../examples/pos_features.py < pos.test.txt > pos-enhanced.test.txt
```

And look at the result:
```
head pos-enhanced.train.txt
Confidence N Y N C Co Con Conf e ce nce ence NN
in N N N i in __nil__ __nil__ n in __nil__ __nil__ IN
the N N N t th the __nil__ e he the __nil__ DT
pound N N N p po pou poun d nd und ound NN
is N N N i is __nil__ __nil__ s is __nil__ __nil__ VBZ
widely N N N w wi wid wide y ly ely dely RB
expected N N N e ex exp expe d ed ted cted VBN
to N N N t to __nil__ __nil__ o to __nil__ __nil__ TO
take N N N t ta tak take e ke ake take VB
another N N N a an ano anot r er her ther DT
```

## The new template file ##

```
cat > pos-enhanced.template <<EOF
U01:%x[0,1]
U02:%x[0,2]
U03:%x[0,3]
U04:%x[0,4]
U05:%x[0,5]
U06:%x[0,6]
U07:%x[0,7]
U08:%x[0,8]
U09:%x[0,9]
U10:%x[0,10]
U11:%x[0,11]

U13:%x[0,0]
U14:%x[-1,0]
U15:%x[1,0]
U16:%x[-2,0]
U17:%x[2,0]
U18:%x[-2,0]/%x[-1,0]
U19:%x[-1,0]/%x[-0,0]
U20:%x[0,0]/%x[1,0]
U21:%x[1,0]/%x[2,0]
U22:%x[-2,0]/%x[-1,0]/%x[0,0]
U23:%x[-1,0]/%x[0,0]/%x[1,0]
U24:%x[0,0]/%x[1,0]/%x[2,0]
U25:%x[-2,0]/%x[-1,0]/%x[0,0]/%x[1,0]
U26:%x[-1,0]/%x[0,0]/%x[1,0]/%x[2,0]
U27:%x[-2,0]/%x[-1,0]/%x[0,0]/%x[1,0]/%x[2,0]

B
EOF
```

There are three groups of features. First, we add one template for each of the eleven new features. This results in `U01%x[0,1]` to `U10%x[0,11]` which represent the zero-index fields of the new features. Notes that they are prefixed with numbers in order to differentiate features which would end up having the same value (ex: the word "to" in our previous example has the same value than the two-character-prefix feature "to"). The second group of features generates word n-gram features using column zero and up to two words before the current word (`%x[-2,0]`) and two words after (`%x[2,0]`). In our previous example, when processing the word `pound`, the template `U26:%x[-1,0]/%x[0,0]/%x[1,0]/%x[2,0]` generates `U26:the/pound/is/widely`.

The `B` feature is defined again to keep context when performing predictions. We could also extend it for word n-grams, but it would largely increase the number of weights and therefore the memory needed for training.

## Training the model ##

```
../mira -t -f 2 -s 0.01 -n 10 pos-enhanced.template pos-enhanced.train.txt pos-enhanced.model pos-enhanced.test.txt
read 27 templates from "pos-enhanced.template"
counting: 8936
unigrams: 1647979, bigrams: 1
unigrams: 222066, bigrams: 1, cutoff: 2
labels: 44
model: 9772840 weights
iteration 0
  train: 8936 examples, terr=0.0698 fscore=0.9302
  test: 2012 examples, terr=0.0501 fscore=0.9499 (ok=45005 ref=47377 hyp=47377)
iteration 1
  train: 8936 examples, terr=0.0285 fscore=0.9715
  test: 2012 examples, terr=0.0374 fscore=0.9626 (ok=45603 ref=47377 hyp=47377)
iteration 2
  train: 8936 examples, terr=0.0187 fscore=0.9813
  test: 2012 examples, terr=0.0330 fscore=0.9670 (ok=45815 ref=47377 hyp=47377)
iteration 3
  train: 8936 examples, terr=0.0143 fscore=0.9857
  test: 2012 examples, terr=0.0283 fscore=0.9717 (ok=46038 ref=47377 hyp=47377)
iteration 4
  train: 8936 examples, terr=0.0106 fscore=0.9894
  test: 2012 examples, terr=0.0261 fscore=0.9739 (ok=46141 ref=47377 hyp=47377)
iteration 5
  train: 8936 examples, terr=0.0087 fscore=0.9913
  test: 2012 examples, terr=0.0252 fscore=0.9748 (ok=46184 ref=47377 hyp=47377)
iteration 6
  train: 8936 examples, terr=0.0072 fscore=0.9928
  test: 2012 examples, terr=0.0236 fscore=0.9764 (ok=46258 ref=47377 hyp=47377)
iteration 7
  train: 8936 examples, terr=0.0063 fscore=0.9937
  test: 2012 examples, terr=0.0226 fscore=0.9774 (ok=46307 ref=47377 hyp=47377)
iteration 8
  train: 8936 examples, terr=0.0051 fscore=0.9949
  test: 2012 examples, terr=0.0221 fscore=0.9779 (ok=46328 ref=47377 hyp=47377)
iteration 9
  train: 8936 examples, terr=0.0052 fscore=0.9948
  test: 2012 examples, terr=0.0219 fscore=0.9781 (ok=46340 ref=47377 hyp=47377)
writing model: pos-enhanced.model
```

Here, we can see that we get an accuracy of 97.8 on the test set. This is very nice, and probably almost the state of the art on that dataset.

## Tagging some real text ##

Real text does not come one word per line. You need to tokenize it in order to get words that your tagger can recognize. To match the training data that we have used, you can get the Penn Treebank tokenizer script:
```
wget http://www.cis.upenn.edu/~treebank/tokenizer.sed
```

Then, write some text, tokenize it, put one word per line, generate features and finally run the part-of-speech tagger on it.
```
echo "I'd do that if I could." | sed -f tokenizer.sed | tr " " "\n" | python pos_features.py | ../mira -p pos-enhanced.model 
reading model: pos-enhanced.model
I N N N I __nil__ __nil__ __nil__ I __nil__ __nil__ __nil__ PRP
'd N N Y ' 'd __nil__ __nil__ d 'd __nil__ __nil__ MD
do N N N d do __nil__ __nil__ o do __nil__ __nil__ VB
that N N N t th tha that t at hat that IN
if N N N i if __nil__ __nil__ f if __nil__ __nil__ IN
I N N N I __nil__ __nil__ __nil__ I __nil__ __nil__ __nil__ PRP
could N N N c co cou coul d ld uld ould MD
. N N Y . __nil__ __nil__ __nil__ . __nil__ __nil__ __nil__ .
```

The predicted tags are at the end of each line. You can check the intermediate results to get an idea of how each step works.

# Live demo #

&lt;wiki:gadget url="http://miralium.googlecode.com/svn/trunk/demo/tagger-gadget.xml" border="0" width="1200" height="200"/&gt;

# Going further #

Here are some directions you can explore:
  * What happens if you don't use the `B` feature?
  * Can you come up with the set of features and training parameters (`-f`, `-s`, `-n`) that produce the best result? Is there something wrong in checking the error on the test set while searching for those parameters?
  * Use a subset of the training set as development set for parameter tuning and check the error on the test set. Do you get the same results?
  * How does the order of training examples affect mira?
  * Implement the morphological features in java and make a java-only POS-tagger.