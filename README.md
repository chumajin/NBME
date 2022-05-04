

First of all, I would like to thank kaggle and the staff for hosting such an interesting competition.
Also, I really appreciate my teammates, @harshit92 , @laplaceplanet , @xbustc . It was really nice to be in a team with you guys.


# 1. Summary 
1.1 The main part of our solution is pre-training with pseudo-labels.

1.2 Pseudo-labels were used not only to use 1.1, but also to increase training data.

1.3 
- submission1 made **LB** the highest. 

(public score 0.893: Public rank10th but private score 0.892 is not good.)


- submission2, we chose the one with the highest **CV**. 

(public score 0.893: Public rank10th but private score 0.892 is better.now private rank is 14th.)

 **Trust cv** is very important. This strategy is referred by @cdeotte [PetFinder solution](https://www.kaggle.com/competitions/petfinder-pawpularity-score/discussion/301015).

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Final submission2 model: Ensemble weights were all 0.25.

- deberta v3 large (pre-training with pseudo-labels) lb 0.889
- electra (pre-training with pseudo-labels) lb 0.886
- deberta v3 large (increase training date with pseudo-labels) lb 0.888
- deberta v1 xlarge (pre-training with pseudo-labels & increase training date with pseudo-labels) lb 0.889

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

------------------Details below-------------

# 2. Model making

My model is created by merging the @abhishek [code](https://github.com/abhishekkrthakur/long-text-token-classification) made in Feedback competition and @yasufuminakama [code](https://www.kaggle.com/code/yasufuminakama/nbme-deberta-base-baseline-train).

Main tips are as follows.

- mixup @shujun717 [code](https://www.kaggle.com/competitions/feedback-prize-2021/discussion/313235)
- random mask
- multi sample drop out
- 5 times validation in one epoch
- 8bit adam @nbroad [code](https://www.kaggle.com/code/nbroad/8-bit-adam-optimization/notebook?scriptVersionId=85409379)
- gradient checkpoint
- change lr between backborn and head
- non-use maxlen of tokenizer function. (just align the size in each batch)
- sort validation data by the length of the sentence

Our best model without pseudo label is lb 0.887 of deberta v1 xlarge, 0.887 of deberta v2 xlarge.

# 3. Pseudo labeling

## 3.0 how to avoid the leakage from pseudo-labeling

As already reported in other solutions and forums, misuse of pseudo-labels will cause a leak in this competition. Specifically, if there are a lot of leaks, the cv will be 0.9 or higher.

This is likely to occur if there is text in the pseudo-label that is similar to the validation text.

Therefore, to avoid this, the pseudo-label should not be calculated using the average of the predictions, and should be used as is without the average.


![](https://github.com/chumajin/NBME/raw/main/leak1.jpg)

![](https://github.com/chumajin/NBME/raw/main/leak2.jpg)




## 3.1 Method 1 : pre-training with pseudo labeling(Public LB~+0.003)

This is @xbustc 's great idea. He made the strong model for us as follows.

Step 1 : normal training from train.csv and making models.

Step 2 : inference 612602 ids from patience notes in order to make pseudo labeling

Step 3 : 50% of pseudo label date made from Step2 was used as train data.
       And just 1 epoch training was done

Step 4 : He trained 14300 ids from train.csv again by loading weights made in  Step3.

As a result, deberta v3 large is boosted from lb 0.886 to 0.889.
electra is boosted from lb 0.883 to 0.886. These models are very strong when we ensemble them.

## 3.2 Method 2 : increase training date with pseudo-labels(Public LB~+0.004)

This is just increasing the training data in my case.

Step 1 : normal training from train.csv and making models.

Step 2 : inference 14300×5 ids from patience notes in order to make pseudo labeling

Step 3 : For each fold, I used different pseudo-labels to add.
The 14300 pseudo label in each were added to the each train data 
(all train ids in 1 epoch = 14300 ids × 4/5 from train.csv and 14300 from pseudo labels).  

This makes a deberta v3 large boosted from lb 0.884(cv 0.8869) to 0.888(cv 0.89084).
And a deberta v1 large boosted from lb 0.887(cv 0.8871783 ) to 0.888(cv 0.890706)

## 3.3 The combination of #3.1 and #3.2 (~ more Public LB +0.001 )

This is just the combination of Method 1 and 2.

And a deberta v1 large boosted from lb 0.888(cv 0.890706) to 0.889(cv 0.89112)


# 4. Ensemble

# 4.1 charpred ensemble

We use the charpred ensemble. We tried the Weighted Box Fusion(WBF). But charpred was better. This is because the resolution of prediction signal is higher than WBF in our case and the target of this competition is the char location.

# 4.2 model selection

To be honest, We had the hardest time here. After the rank of our team went up to 2, it was necessary to select a model by cv in order to avoid a big shake down and better shake up. We spent a lot of time there. (ex. making the leak-free models and so on..) But, we couldn't have a complete pseudo leak-free model. However, we had to choose the optimal solution from these models.

Example of models.
- deberta v1 xlarge, v2 xlarge, v3 xlarge, large-mnli
- roberta large
- electra

Example of methods
- using not only BCE but cross-entropy
- remove pseudo samples which pn_history are familiar to train samples
- 2nd around pseudo
- pretrain with pseudo
- increasing with pseudo

Basically, the models were selected by calculating cv using the hill climb method. However, for models that are suspected of leaking even a little, we have set a rule to adopt only those that can improve both cv and lb.

As a result, we were able to select a private score that were optimal to some extent, and we were able to prevent a large shake down.

The one with the best public score was bad for private, so we think we were able to get the correct answer.

# 4.3 ensemble weights
We tried to use the nelder mead method, but since there is a possibility of a leak, we decided not to use. The weights were divided into all 0.25.

## 4.4 final submission
Our final cv was 0.898527 with a little leak of pseudo label.Final models are written in 1.Summary chapter.

# 5. post-process

We just use the @theoviel public [code](https://www.kaggle.com/code/theoviel/roberta-strikes-back). I think there were improvements, as in other people's solutions.

# 6. not working for us

- MLM (other team is working...)
- LSTM on output layer of transformer
- Add feature text at the start, instead of later
- Added class weights based on feature text frequency, and misclassification based on oof data
- Adding new tokens to vocabulary (like mycordial/medical terms)
- lower case of input
- LGBM for post process

# Acknowledgments

We couldn't get this score on our own. Thank you to everyone who shared past knowledge and code! We respect to you.

Special thanks to this competition (using the code, dataset, and strategy)
@abhishek,@cdeotte,@nbroad,@shujun717,@theoviel,@yasufuminakama

# Postscript

As you know, The last cleanup in recent competitions. We haven't reached the gold medal at this point, but we strongly hope to have a healthy ranking in the end.
