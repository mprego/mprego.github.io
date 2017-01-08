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

~~~~
Just fantasy points with past 5 games: 64.7
Just fantasy points with past 5 games: 64.5
All variables with past 5 games: 67.1
All variables with past 10 games: 64.0
~~~~

Based on this data, it looks like there isn't a huge difference among the four models, but the lowest score does belong to the model predicting all stats with the past 10 games.  

#### Choosing the Optimal Fantasy Lineup
Once I had my predictions, I still had a non-trivial task: choose which players to put in my daily fantasy lineup.  On Fanduel, there are two constraints:
1. Position: According to positions defined by Fanduel, my lineup had to consist of 2 PGs, 2 SGs, 2 SFs, 2 PFs, and 1 Center
2. Salary: According to salaries from Fanduel, my lineup had to be at or below the $60,000 daily budget

Given these two constraints and my objective to maximize the number of points, I had to build an algorithm.  Given the complexity of the two constraints, I tried to build an algorithm that chose the best lineup through trial and error.  The easiest and sure-fire way of finding the optimal lineup would've been to test every possible lineup.  This was not feasible, however.  In a typical day, there are about 200 players active, meaning that there's about 50 possible players for each position.  Using combinatinatorial math, we have:

~~~~
Number of Possible Lineups = (50 choose 2) * (50 Choose 2) * (50 Choose 2) * (50 Choose 2) * (50 Choose 1)
Number of Possible Lineups = (1225)^4 * 50
Number of Possible Lineups = 1.13 * 10^14
~~~~

To make this algorithm feasible, I'd need to cut down the potential number of players.  To do this, I only considered a subset of weakly dominant players with respect to fantasy points and salary.  To pick this subset, my logic was:

1. For each position, sort the list of players by predicted fantasy points in descending order
2. Add the first player to the subset
3. Look at the next player on the sorted list.  If the salary was lower than the salary of the player in the subset, add this person to the subset
4. Repeat this comparison until I ran out of players

In essence, my subset was the frontier of players who have the best Salary to Fantasy Points efficiency.  Since every position other than Center requires two selections, I repeated this process after removing the first group of players.  This way, for each position, I would have a subset that included the two most efficient Salary to Fantasy Points players at each Salary level.  Here's what the selection of players looked like for point guards playing on 12/5/2016:

{% include frontier.html %}

Using this strategy to cut down the number of potential players brought down the number of potential lineups.  In this example, I reduced the number of points guards from 51 to 16.  Now we can recalculate the number of possible combinations:

~~~~
Number of Possible Lineups = (16 choose 2) * (18 choose 2) * (21 choose 2) * (16 choose 2) * (6 choose 1)
Number of Possible Lineups = 120 * 153 * 210 * 120 * 6
Number of Possible Lineups = 2.78 * 10^9
~~~~

Although is a large reduction, it'll still take a while to run through every single possible combination.  To speed up the process, I decided to run simulations to pick the best lineup.  In this case, I ran 1,000 simulations and picked the lineup with the highest predicted fantasy points that was at or under budget ($60K for Fanduel).

### Comparing Model Performance vs FanDuel's Model and Historical Data
Now that we have a working model to predict fantasy points and create a fantasy lineup, we can compare how this lineup would've done on Fanduel.  For games from 12/5/2016 to 12/29/2016, I compared my best performing model (the model that predicts all variables with the past 10 games) to Fanduel's own model and to the actual historical data.

For those range of dates, here are the average fantasy points per lineup:

~~~~
Model with all variables using past 10 games: 270.4
Fanduel's predictions: 269.0
Actual: 354.0
~~~~

{% include score_comps.html %}


### Conclusion and Next Steps
The results leave something to be desired.  Compared to the Fanduel's own predictions, there isn't much of a lift with my model.  With such a small difference, the models are virtually tied in terms of performance.  

Compared to the actual data, there is a large gap.  Some gap is always to be expected since the data cannot account for every factor that affects a player's performance.  The gap is so large though, that I question whether the models used are appropriate for this prediction.  To predict a player's fantasy points, we used regression that made sure the model was unbiased and optimized to minimize the MSE for all data points.  In this final comparison, however, we aren't looking at all of the data points; we're only looking at the players that have the highest scores for a given budget.  Since only those players show up in the end result, it may make more sense to fit the model to those high performing outliers.  In that case, overall accuracy doesn't matter as much as the accuracy for those unlikely events.  Predicting those breakout outlier performances can lead to a lineup closer to reality, meaning higher points and better success.
