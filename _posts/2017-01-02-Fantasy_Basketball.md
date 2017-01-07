---
layout: post
title: Fantasy Basketball Predictions
---

Predicting fantasy basketball points using data from Basketball Reference.

Daily fantasy sports has become become increasingly popular, which is great because it's well suited for a data science project.  First, there's tons of repeatable events to learn from and to model.  Second, there's more than enough easily accessible data at various granularities.  Finally, there are solid benchmarks from other competitors and from the daily fantasy sites themselves for how well your predictions fare.

### Data Source
Two sites were used to gather data for this project.  First, Basketball Reference was used to get past game and player data as well as the games for the current day.  Second, FanDuel was used to collect the daily player lineups, which had two valuable pieces of information: salary and position.  These two pieces of data weren't used for predictions, but were necessary for choosing the optimal fantasy lineup.

Scraping data from these sites was difficult because the daily pace of fantasy competitions made it imperative that data was refreshed every day.  To accomplish this workflow, a Luigi pipeline was built that scraped the data daily and updated a file of historical data.  The code used for scraping can be found [here](https://github.com/mprego/NBA_Player/blob/master/src/data/Scraping.py)

### Model Building
In this project, there are two models: one to predict the daily fantasy points per player and one to choose the optimal lineup.  

#### Predicting Fantasy Points
To predict daily fantasy points, I used player-level data from Basketball Reference.  The data included both simple box score metrics, like points, assists, and rebounds, and advanced statistics, like true shooting percentage, assist percentage, and usage rate.  Once I had these variables, I cleansed the data and made ridge regression and gradient boosting regression models.  The exact pipeline looked like this:

1. Scale Variables (center and standardize)
2. Feature Selection (selects k-best features)
3. Create models (Ridge Regression and GBM using various parameters for each model)

For all three steps, I performed a grid search that optimized for a low MSE.  This algorithm allowed the computer to optimize the model without a lot of user involvement.  There were two areas, however, that I tested out manually.

First, I tested how far back to use for past performance.  The possible choices ranged from the entire season to just the last game.  There are pros and cons of these different time ranges.  Having a window too long risked missing recent trends, such as short-terms streaks and also short-term injuries.  Having a window too short risked the model overreacting to standalone performances or slumps.  To balance these tradeoffs, I settled on two periods to test: last 5 games and last 10 games.  I felt both of these periods provided enough data to capture significant trends while ignoring sporadic one-off performances.  

Second, I also tested how many models to build.  In Fanduel's games, fantasy points are a combination of points, steals, assists, rebounds, blocks, and turnovers.  I could either model each of these separately or just model the total points.  Again, the best method isn't clear immediately.  Building models for each variable may result in more accurate models, leading to more accurate fantasy point predictions.  On the other hand, building models for just fantasy points may lead to a more robust model, since many of the stats are not independent, such as points and assists.  

With these two sets of options and the model pipeline described above, I created 4 models and compared the MSE of the fantasy point predictions below:

FD 5: 64.7
FD 10: 64.5
all vars 5: 67.1
all vars 10: 64.0

Based on this data, it looks like there isn't a huge difference among the four models, but the lowest score does belong to the model predicting all stats with the past 10 games.  

### Choosing the Optimal Fantasy Lineup
Once I had my predictions, I still had a non-trivial task: choose which players to put in my daily fantasy lineup.  On Fanduel, there are two constraints:
1. Position: According to positions defined by Fanduel, my lineup had to consist of 2 PGs, 2 SGs, 2 SFs, 2 PFs, and 1 Center
2. Salary: According to salaries from Fanduel, my lineup had to be at or below the $60,000 daily budget

Given these two constraints and my objective to maximize the number of points, I had to build an algorithm.  Given the complexity of the two constraints, I tried to build an algorithm that chose the best lineup through trial and error.  The easiest and sure-fire way of finding the optimal lineup would've been to test every possible lineup.  This was not feasible, however.  In a typical day, there are about 200 players active, meaning that there's about 40 possible players for each position.  Using combinatinatorial math, we have:

~~~~
Number of Possible Lineups = (40 Choose 2) * (40 Choose 2) * (40 Choose 2) * (40 Choose 2) * (40 Choose 1)
Number of Possible Lineups = (780)^4 * 40
Number of Possible Lineups = 1.48 * 10^13
~~~~

To make this algorithm feasible, I'd need to cut down the potential number of players.  To do this, I only considered a subset of weakly dominant players with respect to fantasy points and salary.  To pick this subset, my logic was:

1. For each position, sort the list of players by predicted fantasy points in descending order
2. Add the first player to the subset
3. Look at the next player on the sorted list.  If the salary was lower than the salary of the player in the subset, add this person to the subset
4. Repeat this comparison until I ran out of players

In essence, my subset was the frontier of players who have the best Salary to Fantasy Points efficiency.  Since every position other than Center requires two selections, I repeated this process after removing the first group of players.  This way, for each position, I would have a subset that included the two most efficient Salary to Fantasy Points players at each Salary level.  Here's what the selection of players looked like for point guards playing on 12/5/2016:

{% include frontier.html %}

Using this strategy to cut down the number of potential players brought down the number of potential lineups, but not by enough to test every one.  Even though it wouldn't be 100% accurate nor efficient, I decided to run simulations to pick the best lineup.  

(end here)

### Variable Selection
To use a team's Four Factors in past games as a variable, I needed to decide how many past games to consider.  On one hand, picking a lot of past games would help reduce the variance of this variable for a team.  But a high number of past games would also restrict the eligible games for the model to predict as the first few games of the season would not have enough past games.  On the other hand, picking a smaller number of past games could capture short-term swings in a team's performance and allow the model to predict more games earlier in the season.  To determine the exact number, I tested these variables when cross-validating the model.

I also added a few other variables that seemed to make intuitive sense. :

- Back to Back Games: Teams often are tired in their second game of a back-to-back schedule, causing them to struggle
- Win % for last 5 and 15 games: This metric doesn't add a ton of information over the Four Factors, but its inclusion might be able to reveal a team's instincts and willingness to win
- Dunk Score: As mentioned in this [post](http://mprego.github.io/Dunks/), the Dunk Score has some predictive power in predicting game outcomes

Next, I used all of these variables with two types of models: a Support Vector Machine (SVM) model and a Random Forest model.  For classification problems with lots of variables, these models perform well.  Using 10-fold cross validation, I acheived the following accuracies:


Only past 5 game variables:

**SVM**:

~~~~
0.642045454545
{'kernel': 'poly', 'C': 5, 'degree': 1}
~~~~

**Random Forest**:

~~~~
0.632575757576
{'n_estimators': 30, 'criterion': 'entropy'}
~~~~


Only past 15 game variables:

**SVM**:

~~~~
0.660984848485
{'kernel': 'linear', 'C': 1, 'degree': 1}
~~~~

**Random Forest**:

~~~~
0.655303030303
{'n_estimators': 50, 'criterion': 'gini'}
~~~~


Including all past games within the season:

**SVM**:

~~~~
0.673295454545
{'kernel': 'rbf', 'C': 10, 'degree': 1}
~~~~

**Random Forest**:

~~~~
0.648674242424
{'n_estimators': 40, 'criterion': 'gini'}
~~~~


All of the variables:

**SVM**:

~~~~
0.683712121212
{'kernel': 'linear', 'C': 0.5, 'degree': 1}
~~~~

**Random Forest**

~~~~
0.666666666667
{'n_estimators': 40, 'criterion': 'entropy'}
~~~~


Based on these results, it looks like including a variety of past n games produces the most accurate model.  Most of this accuracy appears to be driven by the variables that include all past games.  In fact, the model based on those variables alone nearly achieved the same accuracy as the model with every variable.  
