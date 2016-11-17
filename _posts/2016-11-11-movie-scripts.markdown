---
layout: post
title: "Movie Scripts"
date: 2016-11-11T10:19:57-08:00
---

Let's say I'm a computer and I don't know anything about movies. But I can read movie scripts. My human friend wants
to know which movie to watch. How can I help her?

## Step 1: Inferring movie genres from scripts

Maybe my friend wants to watch an action movie?

<img src="{{ root_url }}/source/images/wolverine.gif" />

If I have a script, how can I tell if it's an action movie?

I collected 827 scripts from IMSDb (link). Then for these same 827 movies, I checked to see if they were tagged
as 'action' movies on MovieLens (more details). I end up with 106 action movies, including:

* Terminator
* Ninja Assassin
* Crank 
* X-Men Origins: Wolverine

There also 721 non-action movies, including:

* Mary Poppins
* Gandhi
* Duck Soup

For each of these 827 movies, we want to build a model that uses the script to correctly predict if they are action movies.
The exciting part is that then we can apply the same model to new movies that we don't know anything about to figure
out if they should be called action movies.

### Bag of words

Bag of words means that we only care about the words that appear in the script and the number of time each word appear.
We do not care about the order of words at all.

In other words, these three sentences are all equivalent according to bag of words:

* Houston, we have a problem.
* We have a problem, Houston.
* A we Houston problem have.


Although we lose a lot of information about the movie by doing this, it makes it easier for the computer to analyze.

### Random forest


[Random forests](https://en.wikipedia.org/wiki/Random_forest) are a variant of the basic decision tree method.
The following is a simplified example of what a single decision tree would look like using bag-of-words:

<img src="{{ root_url }}/source/images/decision_tree.png" />

A random forest consists of a collection of many decision trees (here we use up to a thousand decision trees in a single model).
Each tree is random in that the words used both the words used to distinguish and the movies used to train the tree are
randomly selected.

The computer learns the correct decision trees by examining the training dataset (the 827 movie scripts for which we have
both scripts and action tags available).

### Results

The computer reads in all 827 movie scripts, then performs lemmatization on each word, which mean that each word is
transformed to the headword you would find in a dictionary (for example, "explodes" is transformed into "explode"
and "explosions" is transformed into "explosion").

After lemmatization, we find the 10,000 most common words found across all the movie scripts. These words will be used
as predictors.

At this point, every movie has been tagged by a human as either action or non-action. We also have 10,000 predictors, which
are just counts of how many times each word has appeared in the script. We train a random forest model to predict the
action tag based on the word counts.

We can use [cross-validation](https://en.wikipedia.org/wiki/Cross-validation_(statistics)) to estimate how accurate the model will be on data that it hasn't seen yet.

The cross-validated accuracy is 92%.
This means that if we get a new movie where we're not sure if it's an action movie,
we will corrent label it 92% of the time.
92% sounds pretty high. But wait! Only 13% of our movies were action movies to begin with.
That means that if we predicted that every movie was non-action, we would already have 87% accuracy.
Let's look at the cross-validated results in more detail:

* Accuracy rate for movies that are truly non-action: 99% (712/721)
* Accuracy rate for movies that are truly action: 46% (49/106)

Let's look at our predictions for the movies listed above:

Action movies:

* Terminator: <span style="color: green">action</span>
* Ninja Assassin: <span style="color: red">not-action</span>
* Crank: <span style="color: red">not-action</span>
* X-Men Origins: Wolverine: <span style="color: green">action</span>

Non-action:

* Mary Poppins: <span style="color: green">not-action</span>
* Gandhi: <span style="color: green">not-action</span>
* Duck Soup: <span style="color: green">not-action</span>

Is there a way to correctly classify more action movies? Yes, but only if we are willing to misclassify more non-action movies.
This trade-off is shown on a ROC curve:

<img src="{{ root_url }}/source/images/roc_curve_action.png" />

This shows that we can correctly predict about 80% of action movies, if we are willing to misclassify about 10% of non-action movies.

#### Action words

What words are most important for determining if a movie is an action movie or not?

<img src="{{ root_url }}/source/images/wordcloud_action.png" />

<!--
1. weapon (0.076)
1. explodes (0.046)
1. aim (0.046)
1. crash (0.036)
1. fire (0.030)
1. bullet (0.030)
1. wall (0.030)
1. gun (0.027)
1. drop (0.026)
1. chopper (0.026)
1. oh (0.024)
1. steel (0.024)
1. rooftop (0.024)
1. trigger (0.024)
1. air (0.023)
1. fly (0.023)
1. punch (0.023)
1. slam (0.022)
1. building (0.021)
1. blast (0.021)
1. slide (0.021)
1. dive (0.020)
1. helicopter (0.019)
1. control (0.019)
1. scan (0.018)
-->


### What about comedy?

The bag-of-words model is not perfect at predicting action movies, but it does recognize salient features (words
having to do with violence and fast motion) that help identify action movies. It turns out that it is not
equally good at predicting comedies, which make up 8% of our dataset.

<img src="{{ root_url }}/source/images/roc_curve_comedy.png" />

If we want to correctly predict 80% of comedies, we will have end up mispredicting about 40% of non-comedies.
Why is this so bad? Computers aren't good at telling jokes. These are the words most predictive of comedy:

<img src="{{ root_url }}/source/images/wordcloud_comedy.png" />

Not very funny, huh? I guess the jokes are hidden in the order of the words, which bag-of-words explicitly ignores,
rather than in the words themselves. Also, perhaps the genre of comedy is broader than action. The words that identify
a romantic comedy might be different than the words that identify a dark comedy.

So it turns out that bag-of-words is better at identifying certain aspects of movies than others.

<!--
1. guy (0.116)
1. really (0.077)
1. hey (0.074)
1. just (0.072)
1. ball (0.065)
1. married (0.061)
1. asshole (0.056)
1. piss (0.052)
1. dude (0.051)
1. wow (0.048)
1. ups (0.045)
1. jay (0.043)
1. awesome (0.038)
1. chick (0.036)
1. ooh (0.033)
1. cheat (0.031)
1. puke (0.029)
1. mime (0.027)
1. dana (0.023)
1. psychotic (0.022)
-->


## Part 2: Predict rating for many users based on scripts

### Collaborative filtering

If we're in the business of recommending movie, why not go all out and make personalized predictions for the whole
world, predicting how much each person will like every movie ever made. One technique for doing this is collaborative filtering.

Collaborative filtering uses 

Cold-start problem.

The MovieLens dataset.

### The factors
The most important factors come first: 

Let's look at Factor 1.

These movies are the most "Factor 1"

1. Taxi Driver
1. Fargo
1. Trainspotting
1. Being John Malkovich
1. Ed Wood
1. Apocalypse Now
1. Reservoir Dogs
1. Chinatown
1. Rushmore
1. Brazil

<!--
735 	Taxi Driver (1976) 	0.054519
273 	Fargo (1996) 	0.054460
763 	Trainspotting (1996) 	0.052085
100 	Being John Malkovich (1999) 	0.050677
253 	Ed Wood (1994) 	0.049164
62 	Apocalypse Now (1979) 	0.047285
624 	Reservoir Dogs (1992) 	0.046220
172 	Chinatown (1974) 	0.044936
641 	Rushmore (1998) 	0.044562
138 	Brazil (1985) 	0.044533
-->

These are the least "Factor 1":

1. Pearl Harbor
1. Independence Day
1. Entrapment
1. Wild Wild West
1. Air Force One
1. Judge Dredd
1. Cliffhanger
1. Swordfish
1. Runaway Bride
1. Rock, The

<!--
  title   X1
  574   Pearl Harbor (2001)   -0.041965
  396   Independence Day (a.k.a. ID4) (1996)  -0.040441
  260   Entrapment (1999)   -0.038511
  809   Wild Wild West (1999)   -0.035819
  27  Air Force One (1997)  -0.034829
  423   Judge Dredd (1995)  -0.033260
  182   Cliffhanger (1993)  -0.032739
  727   Swordfish (2001)  -0.032206
  638   Runaway Bride (1999)  -0.032128
  632   Rock, The (1996)  -0.032044
-->

It looks like Factor 1 mean something like straightforward vs. dark/devious (cookie-cutter vs. unorthodox).

What words go along with this?

Words:

<!--
[('talkin', 0.00049941776058523384),
 ('anybody', 0.00023369200675495399),
 ('cigarette', 0.00017326107800195019),
 ('heard', 0.00014552169346077515),
 ('actor', 0.00014063436092495967),
 ('drink', 0.000132289254922266),
 ('sit', 6.8177650411176707e-05),
 ('people', 4.901241018706896e-05),
 ('film', 3.9650797512843262e-05),
 ('yeah', 3.0500491697645371e-05),
 ('voice', 1.3952571159930322e-05),
 ('don', 1.1400841228266626e-05),
 ('shot', 4.8403445469470943e-06),
 ('rush', -5.2477076828189414e-07),
 ('telephone', -2.4199103840131539e-06),
 ('monitor', -4.3746852922391439e-06),
 ('know', -4.7065997756649299e-06),
 ('data', -1.3741748686698268e-05),
 ('eye', -1.5390077649593586e-05),
 ('air', -2.210105908904655e-05),
 ('race', -2.4963317757960597e-05),
 ('catch', -2.6498894121412041e-05),
 ('beat', -3.8415830373639563e-05),
 ('drop', -4.9557849293988835e-05),
 ('cell', -6.1547752890865048e-05),
 ('pilot', -6.8315450340073228e-05),
 ('grab', -6.9319411892041245e-05),
 ('leap', -6.9691463100047575e-05),
 ('dive', -7.1194759732302673e-05),
 ('tumble', -7.3387405546546412e-05),
 ('breath', -7.9292930229728559e-05),
 ('rip', -7.9554236775907655e-05),
 ('tech', -8.086076076897151e-05),
 ('team', -8.3656189451853768e-05),
 ('video', -8.4836321142913652e-05),
 ('computer', -0.0001176391203037616),
 ('fly', -0.0001194948753814878),
 ('launch', -0.00018509962592946043),
 ('save', -0.0001919815705980868),
 ('hears', -0.00020477116614011571),
 ('spark', -0.00020489533999584544),
 ('whirl', -0.00021541896805040448),
 ('stun', -0.00022648888264348188),
 ('system', -0.0002393984665033794),
 ('wham', -0.00032399389166426869),
 ('chute', -0.00034764058460298871),
 ('realize', -0.00037926498798270209),
 ('laptop', -0.00039806332550483992),
 ('perfect', -0.00047072969810565713),
 ('fireball', -0.0010288025497466071)]
 -->

Factor 2:

Movies:

Most:

1. Sleepless in Seattle
1. Shakespeare in Love
1. Little Mermaid, The
1. Mary Poppins
1. Pretty Woman
1. Big
1. Finding Nemo
1. Toy Story
1. It's a Wonderful Life
1. American President, The

<!--
 	title 	X2
680 	Sleepless in Seattle (1993) 	0.071983
660 	Shakespeare in Love (1998) 	0.064855
460 	Little Mermaid, The (1989) 	0.060864
494 	Mary Poppins (1964) 	0.060746
597 	Pretty Woman (1990) 	0.057963
105 	Big (1988) 	0.057117
284 	Finding Nemo (2003) 	0.053766
760 	Toy Story (1995) 	0.053395
410 	It's a Wonderful Life (1946) 	0.052090
46 	American President, The (1995) 	0.051890
-->

Least:

1. Natural Born Killers
1. Fight Club
1. From Dusk Till Dawn
1. Fear and Loathing in Las Vegas
1. Event Horizon
1. Trainspotting
1. Big Lebowski, The
1. Pulp Fiction
1. American Psycho
1. Reservoir Dogs

Factor 2 seems to run along a scale from upbeat movies to disturbing movies.

<!--
 	title 	X2
536 	Natural Born Killers (1994) 	-0.075095
282 	Fight Club (1999) 	-0.064130
298 	From Dusk Till Dawn (1996) 	-0.060262
277 	Fear and Loathing in Las Vegas (1998) 	-0.059811
267 	Event Horizon (1997) 	-0.059577
763 	Trainspotting (1996) 	-0.052934
108 	Big Lebowski, The (1998) 	-0.052570
604 	Pulp Fiction (1994) 	-0.051385
47 	American Psycho (2000) 	-0.051318
624 	Reservoir Dogs (1992) 	-0.050662
-->

The factors may be interesting on their own, but can we associate them with the words in the script.

<img src="{{ root_url }}/source/images/factor4.png" />


Words:

<!--
[('spell', 0.00096135956133532811),
 ('marry', 0.00074032471136620118),
 ('philadelphia', 0.00031034575519908888),
 ('tail', 0.00020167067958881351),
 ('thank', 0.00013952040909696132),
 ('potter', 0.00013335462886857445),
 ('word', 0.00011382600527726387),
 ('wait', 4.309378828938053e-05),
 ('nod', 2.055169436982447e-05),
 ('come', 1.1165858541793294e-05),
 ('ask', 1.0466835403040762e-05),
 ('oh', 2.8686617577435966e-07),
 ('hey', 1.646471691640384e-07),
 ('bitch', -0.0),
 ('say', 0.0),
 ('potion', 0.0),
 ('personal', 0.0),
 ('sir', 0.0),
 ('majesty', 0.0),
 ('ll', 0.0),
 ('broom', 0.0),
 ('hand', -0.0),
 ('face', -0.0),
 ('day', 0.0),
 ('mayhem', -0.0),
 ('excuse', 0.0),
 ('wizard', 0.0),
 ('int', 0.0),
 ('door', -0.0),
 ('human', -0.0),
 ('box', 0.0),
 ('shit', -0.0),
 ('clothes', 0.0),
 ('arm', -0.0),
 ('sits', 0.0),
 ('mouth', -0.0),
 ('gun', -1.3426769121932355e-05),
 ('fucker', -2.5624810879245302e-05),
 ('fuckin', -3.6544302144186008e-05),
 ('blood', -5.1755864293955783e-05),
 ('body', -6.263413035288846e-05),
 ('fuck', -0.00010149164624789543),
 ('lime', -0.00013607722801156342),
 ('scream', -0.0001363062747765564),
 ('frame', -0.00015894790540263342),
 ('motherfucker', -0.00024998930142487957),
 ('denny', -0.00027907419343772182),
 ('flesh', -0.0003170184607205796),
 ('pussy', -0.00060992470408107781),
 ('yuppie', -0.00088256272736631078)]
 -->

### How does it do

So we can predict factors based on movie scripts. But what we really want to do is to predict how
each user will rate a given movie. We can do this by using our predicted factors together with
the users 

RMSES:
Predicting all ratings as the global average: 1.006
Predicting all ratings as user average: 0.914
Predicting all ratings using factors, centers, and scales as predicted: 0.907
Predicting all ratings using combination of user and movie average: 0.845
Predicting all ratings using factors, true centers and scales: 0.804
Predictions using full softImpute: 0.661
