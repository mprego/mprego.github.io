---
layout: post
title: Fantasy Basketball Predictions
---

Predicting fantasy basketball points using data from Basketball Reference.

Daily fantasy sports has become become increasingly popular, which is great because it's well suited for a data science project.  First, there's tons of repeatable events to learn from and to model.  Second, there's plenty of easily accessible data at various granularities to build the models.  Finally, there are solid benchmarks from other competitors and from the daily fantasy sites themselves for determining how well your predictions fare.

In this project, I'll build a model that automatically chooses the optimal daily fantasy lineup for the NBA.


### Data Source
Two sites were used to gather data for this project.  First, Basketball Reference was used to get past game and player data as well as the games for the current day.  Second, FanDuel was used to collect the daily player lineups, which had two valuable pieces of information: salary and position.  These two pieces of data weren't used for predictions, but were necessary for choosing the optimal fantasy lineup.

BeautifulSoup was used to scrape data from Basketball Reference and that code can be found [here](https://github.com/mprego/NBA_Player/blob/master/src/data/Scraping.py).  Scraping data from Fanduel was more challenging since the website requires a login and multiple clicks to access the daily player lineups.  To log in and click through the site, I used Selenium, and the code can be found [here](https://github.com/mprego/NBA_Player/blob/master/src/data/Scrape_FD.py).  Finally, all of this scraping had to occur on a daily since Fanduel doesn't have historical daily player lineups.  Since I had to run this code everyday, I built a Luigi pipeline to facilitate errors and dependencies within the process.  The code for that can be found [here](https://github.com/mprego/NBA_Player/blob/master/src/Luigi/BBRef_module.py).


### Model Building
In this project, there are two models: one to predict the daily fantasy points per player and one to choose the optimal lineup.  

#### Predicting Fantasy Points
To predict daily fantasy points, I used player-level data from Basketball Reference.  The data included both simple box score metrics, like points, assists, and rebounds, and advanced statistics, like true shooting percentage, assist percentage, and usage rate.  Once I had these variables, I cleansed the data and made ridge regression and gradient boosting regression models.  The exact pipeline looked like this:

1. Scale Variables (center and standardize)
2. Feature Selection (selects k-best features)
3. Create models (Ridge Regression and GBM using various parameters for each model)

For all three steps, I performed a grid search that optimized for a low MSE.  This algorithm allowed the computer to optimize the model without a lot of user involvement.  There were two areas, however, that I tested out manually.

First, I tested how far back to use for past performance.  The possible choices ranged from the entire season to just the last game.  There are pros and cons of these different time ranges.  Having a window too long risked missing recent trends, such as short-terms streaks and also short-term injuries.  Having a window too short risked the model overreacting to standalone performances or slumps.  To balance these tradeoffs, I settled on two periods to test: last 5 games and last 10 games.  I felt both of these periods provided enough data to capture significant trends while ignoring sporadic one-off performances.  

Second, I also tested how many models to build.  On Fanduel, fantasy points are calculated using this formula:

- _Fanduel Fantasy Points = Points + 1.2*Rebounds + 1.5*Assists + 2*Blocks + 2*Steals -Turnovers_

I could either model each of these components separately or just model the total fantasy points.  Again, the best method isn't clear immediately.  Building models for each component may result in more personalized models for each player, leading to more accurate fantasy point predictions.  On the other hand, building models for just fantasy points may lead to a more robust model, since many of the components are not independent.  For example, if a player scores an unusually higher number of points, their assist numbers may suffer.

I also had to decide whether to build personalized models for each player or one general model for all players.  With the personalized models, I could capture more individual behavior and insights into the model.  This could lead to more accurate predictions, especially for players with lots of data.  The downside, however, is that there may not be enough data to build a robust model for every player and that this method requires a lot more processing power to build and score multiple models.

With all of these possible options, I decided to make my decision based on real performance.  After building each model described above using data from October 25, 2016 and December 4, 2016, I scored the model on a test sample consisting of games from December 5, 2016 to December 20, 2016.  Here are the results:


Model Description | MSE for FD points
------------ | -------------
General, Predict just FD, 5 games | 78.1
General, Predict just FD, 10 games | 77.4
General, Predict all variables, 5 games | 77.9
General, Predict all variables, 10 games | 78.7
Personalized, Predict all variables, 10 games | 78.5

Despite the large differences in building these models, it looks like there isn't a huge difference among them.  This is somewhat surprising given the wide range in model complexity.  Since it's tough to differentiate the models in this step, I'll test each model out later in the process.

#### Choosing the Optimal Fantasy Lineup
Once I had my predictions, I still had a non-trivial task: choosing which players to put in my daily fantasy lineup.  On Fanduel, there are two constraints:

1. Position: According to positions defined by Fanduel, my lineup had to consist of 2 PGs, 2 SGs, 2 SFs, 2 PFs, and 1 Center
2. Salary: According to salaries from Fanduel, my lineup had to be at or below the $60,000 daily budget

Given these two constraints and my objective to maximize the number of points, I had to build an algorithm.  Given the complexity of the two constraints, I decided to build an algorithm that chose the best lineup through trial and error.  The easiest and sure-fire way of finding the optimal lineup would've been to test every possible lineup.  This was not feasible, however.  In a typical day, there are about 50 possible players for each position.  Using combinatorial math, we have:

~~~~
# of Possible Lineups = (2 PGs) * (2 SGs) * (2 SFs) * (2 PFs) * (1 C)
# of Possible Lineups = (50 choose 2)^4 * (50 Choose 1)
# of Possible Lineups = (1225)^4 * 50
# of Possible Lineups = 1.13 * 10^14
~~~~

To make this algorithm feasible, I'd need to cut down the potential number of players.  To do this, I only considered a subset of weakly dominant players with respect to fantasy points and salary.  To pick this subset, my logic was:

1. For each position, sort the list of players by predicted fantasy points in descending order
2. Add the first player to the subset
3. Look at the next player on the sorted list.  If the salary was lower than the salary of the player in the subset, add this person to the subset
4. Repeat by looking at the next player until I ran out of players

In essence, my subset was the frontier of players who have the best fantasy points-to-salary ratios.  Since every position other than Center requires two selections, I repeated this process after removing the first group of players.  This way, for each position, I would have a subset that included the two most efficient fantasy points-to-salary players at each salary level.  

Here's what the selection of players looked like for point guards playing on 12/5/2016
(blue dots are top frontier, orange dots are second frontier, black dots are rest of players):

{% include frontier.html %}

Using this strategy to cut down the number of potential players brought down the number of potential lineups.  In this example, I reduced the number of points guards from 51 to 16.  Now we can recalculate the number of possible combinations:

~~~~
# of Possible Lineups = (16 C 2) * (18 C 2) * (21 C 2) * (16 C 2) * (6 C 1)
# of Possible Lineups = 120 * 153 * 210 * 120 * 6
# of Possible Lineups = 2.78 * 10^9
~~~~

Although is a large reduction (I reduced the number by 99.998%), it'll still take a while to run through every single possible combination.  To speed up the process, I decided to run simulations to pick the best lineup.  In this case, I ran 1,000 simulations and picked the lineup with the highest predicted fantasy points that was at or under budget.


### Comparing Model Performance vs FanDuel's Model and Historical Data
Now that we have working models to predict fantasy points and create a fantasy lineup, we can compare how this lineup would've done on Fanduel.  For games from 12/5/2016 to 12/20/2016, I compared my best performing model (the model that predicts all variables with the past 10 games) to Fanduel's own model and to actual historical data.

For those range of dates, here are the average fantasy points per lineup:

Model Description | Average FD Lineup Points
------------ | -------------
General, Predict just FD, 5 games | 257
General, Predict just FD, 10 games | 267
General, Predict all variables, 5 games | 252
General, Predict all variables, 10 games | 262
Personalized, Predict all variables, 10 games | 261
Fanduel's predictions | 278
Actual | 359

Here's how two of my best performing models, the generalized model predicting just FD points with 10 games and the personalized model predicting all variables with 10 games compare to Fanduel's predictions and the actual data over the month of December:

{% include score_comps.html %}


### Conclusion and Next Steps
The results leave something to be desired.  Compared to Fanduel's own predictions, my model is equal at best in terms of accuracy and consistency.  My most accurate model (general model predicting just FD points with 10 games) came closest to beating Fanduel's model, but was inconsistent.  My other models tended to be more consistent, but didn't come close to beating Fanduel's model.

Compared to the actual data, there is a large gap regardless of the model.  The gap is so large that I question whether my basic model methodology is appropriate for this prediction.  To predict a player's fantasy points, I used regression that made sure the model was unbiased and optimized to minimize the MSE for all data points.  When selecting the lineup, however, we aren't looking at all of the data points; we're only looking at the players that have the highest scores for a given budget.  Consequently, only a small subset of predictions are vital to picking the best lineup.  Perhaps it makes more sense to build a model that narrowly focuses on this objective, instead of broadly focusing on minimizing error for all players.  

The true test of these models that I was unable to perform was to pit them against real competition.  My best proxy for competition was the lineup achieved using actual data and the general consensus online that achieving 300 fantasy points usually puts you in the top half of the competition.  However, this benchmark is just an assumption and deserves to be tested out as well.  

****
