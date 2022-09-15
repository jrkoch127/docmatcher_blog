```
---
layout: blog_post
title: "ADS Docmatcher"
author: "Jennifer Koch, Golnaz Shapurian, Carolyn Grant, Donna Thompson"
position: "ADS"
category: blog
label: general
thumbnail: blog/images/Docmatching%20Pipeline.jpg
---
```

# ADS Docmatcher

**Goal**: Match preprint records that exist in ADS to their published counterpart, and the reverse. 

The primary aim of the Docmatcher pipeline is to match (arXiv) preprints to the peer-reviewed published version, and vice versa. ADS previously used Classic to accomplish this, however with recent updates and developments to ADS Solr, it has become a high priority to migrate Classic processes to Solr and eliminate Classic dependencies. Therefore, the ADS team has made new updates, tests, and implementations for this new version of the ADS Docmatcher.

## Background & arXiv Relationship
ADS regularly retrieves data from arXiv, a free distribution service and open-access archive for scholarly articles (not peer-reviewed by arXiv). It is a widely used practice in the scientific community to share preprint papers in arXiv before and during the process of peer review from an official publisher or journal. ArXiv began its services in 1991, at which time ADS was beginning to launch its database of astrophysical abstracts. Therefore, it was only natural that ADS implemented a process to retrieve papers from arXiv, as it was also the major source where scientists were able to instantly share their work with the community, faster and ahead of the peer-reviewed journal publication process.

ArXiv releases data [five times per week](https://arxiv.org/help/availability), and lately has averaged about [15k manuscripts per month](https://arxiv.org/stats/monthly_submissions), which makes about 680 manuscripts per release! Therefore, ADS indexes arXiv articles nightly, five times per week, via an OAI harvest, and tries to match incoming articles to those already in ADS. In addition, at the end of ADS weekly physics and astronomy collection updates, an attempt is made to match all new _peer-reviewed_ published papers to unmatched arXiv papers.

## Purpose & Benefits
With so many preprints being released by arXiv, it is necessary that ADS analyze and process this content for multiple reasons:
- We aim to ingest new records for content relevant to ADS collections in order to expand coverage of the scientific literature. 
- We aim to keep ADS collections updated with the peer-reviewed versions that are published, so that users have the most accurate, up-to-date information possible. 
- As a result, we improve citations and metrics, and eliminate double counting of a single scholarly work (preprint + published versions).
Users benefit from this merge; authors prefer their published papers to be cited rather than their preprint. 
- Users also have an opportunity to discover and obtain an open access (preprint) for published articles that are behind a paywall.
- Publishers like this merge because they prefer researchers to use the published version of the record as the definitive source.
- For citation purposes; the matching allows ADS to capture citations made either with an arXiv ID, or with publisher metadata.

## Development
**Problem**: Given text and metadata (DOI, abstract, title, authors, year) of an input record, find a matching record that exists in ADS. If the record is an arXiv paper, the match has to be a publisher record; and the other way around. 
**Solution**: Docmatcher utilizes ADS API services, first by processing the article metadata and sending it to Oracle. Using the Oracle service, it queries Solr, and computes the confidence score of the match(es). This process can be described in the following steps.

1. Docmatcher is given a set of metadata (DOI, abstract, title, authors, year) for a paper, arXiv or otherwise, and sends it to the Oracle Service.
2. Oracle queries Solr with the DOI as available, otherwise the abstract, otherwise the title.
3. Oracle computes the similarity between the input record and the resulting record(s) returned from Solr. This includes computed scores for each of the available data fields.
4. Based on the similarity scores, Oracle determines if the two records are a match, and gives it a confidence score.
5. Oracle sends the bibcode results to Docmatcher for Curation staff to review and verify.

<p align="center"
<div class="text-center">
 <img class="img-thumbnail" alt="Flow diagram of Docmatcher Pipeline" src="https://github.com/jrkoch127/docmatcher_blog/blob/main/Docmatching%20Pipeline.jpg" width="800" align="center" />
</div>
</p> 
<p align="center">
           <em>Diagram of Docmatcher Pipeline</em>
</p>
<br>

**Score Computation Model**:
We trained a deep learning model, initially experimenting with the number of layers and nodes. We also considered the activation function, introducing non-linear complexities to the model (Brownlee, J., “[How to Choose an Activation Function for Deep Learning](https://machinelearningmastery.com/choose-an-activation-function-for-deep-learning/)”, MachineLearningMastery, 2021). For the intermediate layer, we chose the ReLU (Rectified Linear Unit), which basically zeros out negative values. Since this is a binary classification problem and the output is a probability, the output layer is a 1 node with a sigmoid function for the activation.

We experimented with two loss functions: binary cross-entropy (logistic regression) which computes the cross-entropy loss between prediction and target, and MSE (linear regression) to compute the mean squared error between prediction and target. Cross-entropy loss measures the performance of a classification model whose output is a probability value between 0 and 1. It is preferred for classification, while mean squared error is used mostly for regression. The problem here can be looked at from both the classification, belonging to the match class, and the estimation, probability of being a match, and hence both loss functions can be applied.

We also experimented with two optimizer functions: Adam and RMSprop, where both are a variant of stochastic gradient descent. The goal is to tune the learning rate parameter, which tells the optimizer how far in the opposite direction of the gradient to move the weights of the layer. The optimizer parameter is important, since if it is too high then the training of the model may never converge, and if it is too low then the training is very slow.

There were two other parameters to consider for the iteration during training: number of epochs and batch size. We experimented with multiple values for these parameters. By creating and evaluating models with different parameters, we were able to identify the best hyperparameters and selected the model with the highest accuracy. 

## Testing & Verification
While refining the new computation model, we reviewed several data sets of daily docmatching results, verifying the correctness of the matches. The Docmatcher results were provided to Curation staff as Google Sheets that included the following metadata: arXiv ID, Classic bibcode matched, Solr bibcode matched, Oracle’s confidence score and similarity scores, as well as Oracle’s label (Match or Not Match). We manually verified the matches to be sure that the label was correct, and also identified correct bibcodes for any false matches that were found. Having a clean training dataset allowed us to train the best model and as a result improved the scoring accuracy.
 
The training set brought the total number of data points to 1543 (696: Match, and 847: Not Match). We ran the hyperparameter selection script on the data before and after the last set was updated, which showed a slightly higher accuracy in the end.

## Advantages of Docmatching with Oracle & Solr
The main differences between the Docmatcher utilizing Classic vs. Oracle/Solr services is how the arXiv data and the DOI is treated, as well as confidence scoring. 

The advantage of the Docmatcher using ADS API services is that it scores the similarity of abstract, title, author, year. On the Classic side, if there is a DOI, it takes it as a match and accepts it. Now on the API side, we compute the similarity scores of the metadata fields, and the final match is based on the confidence score. There were several common cases where the Oracle/Solr pipeline made improvements over the Classic pipeline in correctly identifying matches:
 
- Based on the DOI provided, Docmatcher analyzes the match as a whole, not just by DOI alone.
- Docmatcher results reveal when a curation improvement is needed to make a match.
- Docmatcher recognizes the difference between a book chapter and the book itself, as the confidence scores reveal a difference in title and abstract.
- Docmatcher now stores historical match data with confidence scores in a Postgres database within Oracle, so that it does not match related papers that have lower scores.

## The Future of Docmatcher
ADS plans to utilize Docmatcher indefinitely to fulfill the needs of scientists and bring its inherent benefits to users engaging with ADS content. We will aim to improve the accuracy as necessary should new cases pop up, and the ADS Curation Team will validate new matches that have low confidence scores. A Curation workflow is in progress that will define the curators’ role in verifying matches.

**Future ADS development ideas inspired by Docmatcher**:
- Incorporate preprints from ESSOAr: Earth and Space Science Open Archive
- Analyze content coverage of NASA STI
- Improve records of PhD theses and/or connect to institutional repositories
- Improve matching of books from Harvard Libraries (HOLLIS)


