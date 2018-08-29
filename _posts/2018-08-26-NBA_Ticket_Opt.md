---
layout: post
title: NBA Season Ticket Optimization
---

See how I how researched and optimized my Washington Wizards partial season ticket plan to maximize expected revenue using a MILP optimizer.

# Overview

The professional sports ticket market is like any other market.  There are sellers, buyers, and traders. Whenever there's supply and demand for a commodity, there's the possibility of of making a profit.  In this post, I'll describe how I optimized my Washington Wizards partial season ticket plan to maximize profit.

# The marketplace

First, it's helpful to know the context of the marketplace for professional sports tickets, specifically for the NBA.  To explain the nuances, I'll go through the commodity (which is the ticket itself), the sellers, and the buyers.

In this marketplace, the commodity is a ticket to a sports game.  To be exact, this commodity actually isn't a commodity by definition.  Each ticket is assigned to a specific seat, so in theory each ticket is unique and valued differently.  In practice, however, we can treat tickets in the same general location as being equal.  Thus, one factor of the demand for a ticket is the location.  For basketball games, there are two dimensions of the location: the distance to the court and the viewing angle of the court (e.g. behind the basket or on the sidelines).  Typically, the most desirable location is close to the court and on the sidelines.  

The other main factor of the demand is the game.  Some games will be more popular than other games.  There are several reasons that drive this differences, the main ones being the opposing team and the day and time of the game.  In general, you'd expect the most popular games are when the opposing team is popular in your area and the game is on a weekend night.  Together, the popularity of the game and the location of the ticket can explain a lot of the variance in ticket prices.

As an aside, there are several other situational factors that can drive ticket prices up or down.  A team's recent performance can cause games to be more or less popular.  In addition, hype around a new player can also drive prices up.  Unlike the other two factors mentioned above, these factors are not easy to predict, so my models and strategy will not pick up on these situational factors.

The buyers that cause this demand differences can be split up into two groups: resellers and attendees.  Resellers will buy tickets with the intention of reselling them, while attendees will buy tickets with the intention of actually attending the game.  For this project, my primary role is a reseller (though I may attend some games as well).

Finally, it's important to quickly describe the sellers as the selling process is relatively complex.  The main seller is the NBA.  The NBA offers tickets through several packages and deals, but I'll just focus on their partial season ticket plans for this post.  For these partial season ticket plans, the Wizards offer 3 plans:
- 6 games, where you can choose each game with few restrictions
- 13 games, where you can choose each game with few restrictions
- 13 games, where they choose all 13 games for you
To go along with these plans, you also get to select a free bonus game, so in essence, they're 7 and 14 game plans.

In the plans where you choose your games, each game's price is determined by its tier and the location of the tickets.  The tier is determined by the Wizards, but roughly corresponds with the popularity of the game.  In addition, there a couple restrictions based on the tier.  In the 6 game plan, you can only have one game in the top tier (i.e. gold).  The bonus game is also limited to a particular tier: red for the 6 game plan and red or white for the 13 game plans.

All in all, the selection of games sets up a nice optimization problem.

# Optimization Problem

The selection of your tickets is a problem that can be solved with multiple integer linear programming (MILP).  We can neatly summarize the problem in mathematical terms:
- Objective function: Maximize profit: x1*profit1 + ... + xn*profitn
- Constraints:
  - Number of games = 7 or 14: x1 + ... + xn = 7 or 14
  - Number of gold games <= 1 for 6 game plans: x1*gold1 + ... + xn*goldn <=1
  - Bonus game must be of a certain type: x1*red1 + ... + xn*redn >=1 (for 6 game)
    x1*red_or_white1 + ... + xn*red_or_white_n >=1 (for 6 game)
where xi is either 0 (don't buy) or 1 (buy) for each home game (of which there are 40*)

* There are usually 41 home games, but one game is being played in London this year

# The need for an optimization algorithm
To solve the problem above, you need some sort of algorithm to help you find the best choice.  If you were to evaluate every single possibility, it would take a decent amount of processing power and time.  Looking at the 13 game plan as an example, here's how many  combinations are possible:
`40C13 = 12,033,222,880`

There isn't an obvious way to cut down these possibilities to a more manageable size, so it's nearly impossible to iterate through every combination.  This is where an algorithm comes into play.  For MILP problems, a commonly used algorithm is called branch and cut, which is algorithm that I leveraged.

# Solving this in python

To set up and solve this problem in python, I leveraged the OR-Tools package from Google.  After installing the necessary python packages, I set up a MILP solver object with the below code:

`solver = pywraplp.Solver('SolveIntegerProblem', pywraplp.Solver.CBC_MIXED_INTEGER_PROGRAMMING)`

I then made the xi variables by creating n integer variables:

```
for i in range(len(game_list)):
        x_list[i] = solver.IntVar(0.0, 1.0, 'x'+str(i))
```

I then used the `solver.Constraint` class to create constraints for all the cases I described above.  

Next, I enter the objective function, which was to maximize profit:

```  
obj = solver.Objective()
    for i in range(len(game_list)):
        obj.SetCoefficient(x_list[i], int(game_list.loc[i, 'Profit']))
    obj.SetMaximization()
```

I then asked the optimization object to solve the problem and received solution values for each xi variable.  A 0 meant not to buy that particular game, while a 1 meant to buy that game.  

# Results

After running this optimization problem for the 3 plans described above, I found the 6 game plan to be most attractive:
| Plan || Profit || ROI |
| Custom 6 Games || $193 || 41% |
| Custom 13 Games || $160 || 18% |
| Preselected 13 Games || $261 || 40% |

Based on these expectations, the custom 6 game plan and the pre-selected 13 game plans seemed the most attractive financially.  In either case, I was expecting to make around 40% back on my initial investment (i.e. the cost of the tickets).  

# Collecting Data
A key element to this analysis is knowing how much I can resell each ticket for.  Using the available data at hand and given the tight timeline of only a few days, I determined that collecting data from Ticketmaster's website for the upcoming Wizards season was my best bet.  On the website, people were already posting their tickets up for sale for the upcoming season.  Looking at each game, I looked at the ticket prices in similar sections that I was interested in buying and 

Following my experience playing daily fantasy basketball, I decided to see if I could compete in hockey.  On the surface, the NHL seems very similar to the NBA.  There are 82 games, the season lasts from Fall to Spring, and the arenas are often the same.  With respect to daily fantasy, however, there are several key differences.  

First, there are many more active players.  In basketball, the vast majority of minutes go to no more than 10 players.  In hockey, there are several lines of players, resulting in nearly 20 players getting significant minutes.  


Second, not all players have the same scoring rubric: goalies and skaters are separate rubrics:

~~~~
Goalies = 12 * Team Win + 8 * Shutout + 0.8 * Saves - 4 * Goals Against
Skaters = 12 * Goals + 8 * Assists + 1.6 Shots on Goal + 0.5 * Powerplay Goal Bonus + 2 * Shorthand Goal Bonus + 1.6 * Blocked Shots
~~~~

Finally, the lineup composition differs.  In hockey, each lineup needs the following positions:

~~~~
1 Goalie
2 Centers
4 Wings
2 Defenders
~~~~

With this information, I developed models for the goalies and skaters.  Since this was my first time attempting to build a model for the NHL, my goal was to beat the predictions that Fanduel provides for free.  To achieve this, here's the process I went through:

## Data
I kept the data pretty simple by limiting the number of features.  I ended up using just the metrics mentioned in the scoring rubric along with the player's team and the opposing team.  For building my model, I used data from the beginning of the 2017-2018 regular season where every player had at least 15 games under their belt, had played in at least 3 games in the last 10 days, and had averaged at least 10 minutes of ice time.  All of this performance and team data was pulled from hockeyreference.  One last variable was the predicted fantasy points from Fanduel.  To make a fair comparison between my model and Fanduel's model, I withheld this variable from my comparison model.  I did, however, add this variable back in to a second model to see its impact.

## Features
For the performance variables, such as goals, assists, etc., I calculated the average during the past 5, 10, and 15 games.  FOr categorical variables, such as team and opposing team, I converted those to dummy variables.  Overall, the features were pretty straightforward for this first iteration of a model.

## Model Selection
For skaters and goalies, I used the respective training dataset to choose and tune a model that predicted Fanduel points.  I performed a grid search on the parameters for GBM and ridge regression models and selected the model with the lower MSE.  For both models (one using Fanduel's predictions and one not using it), the best performing goalie model was a ridge regression model, while the best performing skater model was a GBM.

Once I had my predictions, I had to choose which players to be in my lineup to maximize my fantasy points.  Similar to my work with on the NBA lineups, I reduced the lineup selection to a mathematical problem and used a multiple integer linear programming optimizer to find the best lineup for each day.  

To compare my two models and the Fanduel model, I first looked at the MSE amongst all eligible players that I had in my partial 2017-2018 regular season dataset:

| Model || MSE of all eligible players |
| My Model || 72 |
| Fanduel Model || 71 |
| My Model with Fanduel || 73 |

Overall, the model's didn't differ much.  Technically, the Fanduel model had the lowest MSE, but it's difficult to tell if the difference is significant.  To dig a bit deeper into this, I next looked at the MSE for players that got selected in the lineups:

| Model || MSE of players in lineup |
| My Model || 119 |
| Fanduel Model || 123 |
| My Model with Fanduel || 145 |

Unfortunately, this data also produces strange results.  It appears that my model and the Fanduel model are very close in MSE, but the model that incorporates all the variables of my comparison model plus the fanduel predictions is worse with a higher MSE.  There is some selected bias with these MSE figures since the lineup compositions are influenced by the model's preditions.  Regardless, it's interesting that the model with the most comprehensive features performs the worse.

Next, I looked at how well these models could produce high scoring lineups:
| Model || Average Fanduel Points per Lineup |
| My Model || 123 |
| Fanduel Model || 113 |
| My Model with Fanduel || 124 |

{% include NHL_FD_2018_comp.html %}

Finally, these results appear to make some sense.  The worst performing model is the Fanduel model with the lowest average fanduel points.  My models are slightly better, producing similar point totals that are higher than the Fanduel model.  

It's odd that MSE scores do not align with the best lineups.  It is possible that this noise from a small sample, but it's also possible that MSE is not a great indicator of lineup performance.  As stated before, the lineup selection optimization is highly dependent on predictions, so it's possible that another element of the predictions has a higher impact on the lineup performance.  Without more data, it's hard to tell if this phenomenon is common, but the lack of alignment does support my focus on testing as many models as possible.  



Future convo
- Talk about data used
- Talk about features used
- Talk about models used
- Talk about model performance
- Talk about DFS lineup scores

OLD

My results and expected long term success in playing daily fantasy during the 2017-2018 NBA Season

With another season of the NBA, I once again tried to compete in daily fantasy competitions using a statistical model.  This year, I focused on refining my model and algorithms and on figuring out whether entering these contests is even remotely profitable.  

Following the disappointing performance of my daily fantasy NBA model last year, I was determined to at least beat the predictions that Fanduel provides to everyone on their site.  To achieve this, I made a few small improvements to my model and process:

1. First, I added additional features to my model.  I added categorical variables representing the team the player is on and the  team the player is facing.

2. I built and scored my model on a stricter subset of players.  I limited the set to players who averaged at least 15 minutes per game and who have played at least 2 games in the past 10 days.  This filtered out many players who had inconsistent playing time, who were unlikely to be placed in a lineup anyways.  

3. I improved the lineup optimization algorithm by using a multiple integer linear programming (MILP) algorithm.  Instead of relying on running numerous simulations to find the best lineup, the MILP algorithm can mathematically determine the optimal lineup.  Not only did this result in higher scoring lineups, but the run times also decreased drastically.

With these improvements, I benchmarked my new model's performance against my best model from last year and the predictions that Fanduel provides.  First, I looked at the MSE of fantasy points predictions per player:

| Model || MSE |
| Old Model || 103 |
| Fanduel Model || 104 |
| New Model || 89.8 |

Compared to my older model and the Fanduel model, my new model has a substantially lower MSE.  This is especially rewarding since I was previously unable to beat the MSE of the Fanduel model.  Next, I looked at the average fantasy points per lineup for each model:

| Model || Mean Points per Lineup |
| Old Model || 256 |
| Fanduel Model || 258 |
| New Model || 281 |

Similar to the MSE results above, my new model once again beat my older model and the Fanduel model.  Looking at the this data day by day, my new model appears to consistently beat both models every day:

{% include NBA_FD_2018_comp.html %}

During this NBA season, I also tried to get a better sense of what score is needed to win the competitions.  For several days, I entered either the 50/50 or Double Up non-beginner competitions for typically $1 or $2.  These two competition styles are very similar, where the top ~50% of entrants about double their money while everyone else loses.  Throughout these days, I found that the average lowest winning score was 284, which is a touch above my model's average.  However, when looking at my model's performance day over day, I found that my model would've placed in the top 50% 28/51 times for a win percentage of 55%.  

If that 55% win percentage is indeed true over time, I'd expect to achieve a 10% return on money invested in these competitions:

~~~~
E(Profit on a $x Double Up game) = P(Win) * x - P(Lose) * x
E(Profit on a $x Double Up game) = 0.55 * x - 0.45 * x
E(Profit on a $x Double Up game) = 0.1 * x
~~~~

Based on this performance and expected return, I should try entering more live competitions next year using this model.  The biggest limitation with this analysis is that I only have data on $1-$2 competitions.  To make more serious money, I'd need to enter in more expensive competitions.  The dilemma is that I don't have data for these more expensive competitions.  It's very possible that the more expensive the competition, the more competitive it is.   Next season, I'll need to enter those more expensive competitions to get a better sense of the economics.
****
