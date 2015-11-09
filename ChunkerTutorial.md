# Introduction #

In 2000, the CoNLL task was [syntactic chunking](http://www.cnts.ua.ac.be/conll2000/chunking/). In this tutorial, we train a chunker that can do as well as the systems involved in the eval.

```
mkdir chunker
cd chunker
```

# Downloading the data #

```
wget http://www.cnts.ua.ac.be/conll2000/chunking/train.txt.gz -O - | gunzip > train.txt
wget http://www.cnts.ua.ac.be/conll2000/chunking/test.txt.gz -O - | gunzip > test.txt
```

If you examine the data, you have sentences with one word per line, its part-of-speech tag and a chunk label. The labels are prefixed with "B" for begin, "I" for inside, and an additional label, "O" means that there is no chunk. This notation is often refered to as IOB or BIO.

```
Confidence NN B-NP
in IN B-PP
the DT B-NP
pound NN I-NP
is VBZ B-VP
widely RB I-VP
expected VBN I-VP
to TO I-VP
...
```

We will train a classifier that recognizes the chunk labels given the words and the part-of-speech tags.

# Defining a template #

Features are extracted from the input using a template. That template contains a set of rules that are applied to each line in the input (note that EOF should not appear in the file, it's here to help your shell).

```
cat > chunking.template <<EOF
U01%x[0,0]
U02%x[0,1]
U03%x[0,0]/%x[0,1]
U04%x[-1,0]/%x[0,0]
U05%x[-1,1]/%x[0,1]

B01%x[0,0]
B02%x[0,1]
B03%x[0,0]/%x[0,1]
B04%x[-1,0]/%x[0,0]
B05%x[-1,1]/%x[0,1]

B
EOF
```

If a rule begins by `U`, for unigram, it generates a feature for the current label, it it begins with a `B`, for bigram, it's for the _joint label_ represented by the current and the previous labels. Then, a rule contains free text to identify the feature (two rules that cannot be identified by the free text are treated as if comming from the same bag) and one or more reference to the input `%x[i,j]` where `i` is the relative line number and `j` is the column number starting from zero. So, `%x[0,0]` is the current word and `%x[-1,1]` is the previous part-of-speech tag.

In the previous template definition, we have a few unigram features (current word, current pos tag, both of them, and bigrams of words and tags) and bigram features (similar to unigram features), and the lonely B which generates features for the pairs of consecutive labels regardless of the input.

# Training the classifier #

We can invoke `mira` in training mode (option `-t`), with filtering of features that occur only once (`-f 2`), for ten iterations (`-n 10`) with an update ceiling of 0.01 (`-s 0.01`) so that updates are not too strong. The `-iob` option lets the fscore computation class know that for scoring, labels use the IOB notation.

```
../mira -t -f 2 -n 10 -s 0.01 -iob chunking.template train.txt chunking.model test.txt
read 11 templates from "chunking.template"
counting: 8936
unigrams: 147851, bigrams: 147852
unigrams: 45441, bigrams: 45442, cutoff: 2
labels: 22
model: 22993630 weights
iteration 0
  train: 8936 examples, terr=0.0811 fscore=0.8709
  test: 2012 examples, terr=0.0629 fscore=0.8962 (ok=21401 ref=23854 hyp=23905)
iteration 1
  train: 8936 examples, terr=0.0526 fscore=0.9163
  test: 2012 examples, terr=0.0564 fscore=0.9096 (ok=21789 ref=23854 hyp=24057)
iteration 2
  train: 8936 examples, terr=0.0433 fscore=0.9308
  test: 2012 examples, terr=0.0562 fscore=0.9125 (ok=21873 ref=23854 hyp=24085)
iteration 3
  train: 8936 examples, terr=0.0371 fscore=0.9405
  test: 2012 examples, terr=0.0536 fscore=0.9150 (ok=21832 ref=23854 hyp=23866)
iteration 4
  train: 8936 examples, terr=0.0337 fscore=0.9455
  test: 2012 examples, terr=0.0529 fscore=0.9164 (ok=21876 ref=23854 hyp=23891)
iteration 5
  train: 8936 examples, terr=0.0303 fscore=0.9512
  test: 2012 examples, terr=0.0510 fscore=0.9186 (ok=21961 ref=23854 hyp=23959)
iteration 6
  train: 8936 examples, terr=0.0265 fscore=0.9564
  test: 2012 examples, terr=0.0494 fscore=0.9221 (ok=21979 ref=23854 hyp=23820)
iteration 7
  train: 8936 examples, terr=0.0256 fscore=0.9578
  test: 2012 examples, terr=0.0503 fscore=0.9216 (ok=21955 ref=23854 hyp=23792)
iteration 8
  train: 8936 examples, terr=0.0247 fscore=0.9595
  test: 2012 examples, terr=0.0491 fscore=0.9230 (ok=22001 ref=23854 hyp=23820)
iteration 9
  train: 8936 examples, terr=0.0239 fscore=0.9608
  test: 2012 examples, terr=0.0480 fscore=0.9245 (ok=22044 ref=23854 hyp=23836)
writing model: chunking.model
```

Note that the error rates given on training data are not accurate _per se_ as they are computed _while_ the model is being updated.

# Test time #

First, get the conlleval script:
```
wget http://www.cnts.ua.ac.be/conll2000/chunking/conlleval.txt -O conlleval
```

Then you can run:

```
../mira -p chunking.model < test.txt | perl conlleval 
reading model: chunking.model
processed 47377 tokens with 23852 phrases; found: 23837 phrases; correct: 22042.
accuracy:  95.20%; precision:  92.47%; recall:  92.41%; FB1:  92.44
             ADJP: precision:  76.59%; recall:  71.69%; FB1:  74.06  410
             ADVP: precision:  79.84%; recall:  78.64%; FB1:  79.23  853
            CONJP: precision:  41.67%; recall:  55.56%; FB1:  47.62  12
             INTJ: precision: 100.00%; recall:  50.00%; FB1:  66.67  1
              LST: precision:   0.00%; recall:   0.00%; FB1:   0.00  0
               NP: precision:  92.75%; recall:  92.54%; FB1:  92.64  12394
               PP: precision:  96.47%; recall:  97.24%; FB1:  96.85  4849
              PRT: precision:  73.68%; recall:  66.04%; FB1:  69.65  95
             SBAR: precision:  83.15%; recall:  86.73%; FB1:  84.90  558
               VP: precision:  92.90%; recall:  93.04%; FB1:  92.97  4665
```

So, you get quite good results on the real eval metric. However, these are based on gold-standard part-of-speech tags, which is not very realistic.

# Similar model with CRF++ #
You can compare this result with a model created by CRF++. We use a simpler template (without complex bigram features) because CRF++ is much slower to train.

```
cat > chunking.template_simple <<EOF
U01%x[0,0]
U02%x[0,1]
U03%x[0,0]/%x[0,1]
U04%x[-1,0]/%x[0,0]
U05%x[-1,1]/%x[0,1]
B
EOF
```
Now, perform training (about 1h):
```
crf_learn -f 2 -t chunking.template_simple train.txt chunking.model_simple_crf1
...
iter=136 terr=0.02549 serr=0.31479 act=1000186 obj=18934.20799 diff=0.00003
iter=137 terr=0.02551 serr=0.31535 act=1000186 obj=18932.92608 diff=0.00007
iter=138 terr=0.02545 serr=0.31435 act=1000186 obj=18931.47416 diff=0.00008

Done!3605.31 s
```
And apply to test data:
```
crf_test -m chunking.model_simple_crf1 < test.txt | perl conlleval -d "\t"
processed 47377 tokens with 23852 phrases; found: 23820 phrases; correct: 22129.
accuracy:  95.47%; precision:  92.90%; recall:  92.78%; FB1:  92.84
             ADJP: precision:  77.88%; recall:  73.97%; FB1:  75.88  416
             ADVP: precision:  83.09%; recall:  78.87%; FB1:  80.92  822
            CONJP: precision:  38.46%; recall:  55.56%; FB1:  45.45  13
             INTJ: precision: 100.00%; recall:  50.00%; FB1:  66.67  1
              LST: precision:   0.00%; recall:   0.00%; FB1:   0.00  0
               NP: precision:  93.13%; recall:  92.95%; FB1:  93.04  12398
               PP: precision:  95.78%; recall:  97.65%; FB1:  96.71  4905
              PRT: precision:  72.12%; recall:  70.75%; FB1:  71.43  104
             SBAR: precision:  85.95%; recall:  78.88%; FB1:  82.26  491
               VP: precision:  93.68%; recall:  93.92%; FB1:  93.80  4670
```
The resulting accuracy is not much better even though with the complete template, you should get higher performance.

CRF++ has MIRA training, but each iteration doesn't seem to be much faster than with regular training, and by looking at the code, you'll see that it actually implements a regular perceptron (Collin's averaged perceptron) which does not perform as good as miralium. For instance, if you use the -a MIRA and -m 10 options with the simple template, training 10 iterations requires about 12 minutes and yields an accuracy of 90.15.

# Timing #

On a not so recent machine, here is what we get:
```
/usr/bin/time ../mira -p chunking.model < test.txt > /dev/null
reading model: chunking.model
5.21user 0.41system 0:04.76elapsed 118%CPU (0avgtext+0avgdata 0maxresident)k
0inputs+64outputs (1major+112213minor)pagefaults 0swaps
```

That's about 10k words per second.
And for training:
```
/usr/bin/time ../mira -t -f 2 -n 10 -s 0.01 -iob chunking.template train.txt chunking.model test.txt
...
74.88user 0.91system 1:13.31elapsed 103%CPU (0avgtext+0avgdata 0maxresident)k
0inputs+9064outputs (1major+166190minor)pagefaults 0swaps
```
It's about 50 times faster than CRF++ and 10 times faster than CRF++ with MIRA training (10 iterations).

# Examining the model #

You can convert the binary model to a text format (compatible with CRF++)

```
../mira -c chunking.model chunking.model.txt
reading model: chunking.model
writing text model: chunking.model.txt
```

The file contains five sections separated by a blank line, the first one is a header, then there is the list of labels followed by the templates. Then come the feature ids and the last block contains all the weights. Each feature has one weight for each label (or pair of label for bigram features) which start in the weight array at the feature id.

```
version: 100
cost-factor: 1
maxid: 22993630
xsize: 4

B-NP
B-PP
I-NP
B-VP
I-VP
B-SBAR
O
B-ADJP
B-ADVP
I-ADVP
I-ADJP
I-SBAR
I-PP
B-PRT
B-LST
B-INTJ
I-INTJ
B-CONJP
I-CONJP
I-PRT
B-UCP
I-UCP

U01%x[0,0]
U02%x[0,1]
U03%x[0,0]/%x[0,1]
U04%x[-1,0]/%x[0,0]
U05%x[-1,1]/%x[0,1]
B01%x[0,0]
B02%x[0,1]
B03%x[0,0]/%x[0,1]
B04%x[-1,0]/%x[0,0]
B05%x[-1,1]/%x[0,1]
B

118756 U04investment/trust
598488 U04,/last
620224 U03grants/NNS
629816 U04have/more
996248 U01opening
745734 U04and/Japan
...
```

Note that you can then use this model with CRF++ (`crf_learn --convert model.txt model.crf`). You can also load/convert CRF++ text models (`crf_learn --textmodel ...`) with miralium.