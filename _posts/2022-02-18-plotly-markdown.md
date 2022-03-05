---
title: "Plotly & Markdown"
tags:
  - R
  - API
  - Data Visualization
  - College Football
  - ggplot2
  - plotly
  - markdown
  - GitHub Pages
categories:
  - GitHub Personal Pages
layout: single
classes: wide
header:
  teaser: /assets/images/plotly-markdown/unnamed-chunk-2-1.png
excerpt: "In a previous post, I created a scatterplot using ggplot2 that displayed college football team logos as the markers using geom_image() from the ggimage package."
---



## Images As Markers

In a [previous post](https://jfking50.github.io/oregon-football/), I created a scatterplot using `ggplot2` that displayed college football team logos as the markers using `geom_image()` from the `ggimage` package. I make plots with `plotly` a fair amount, so I wanted to try to figure out how to do the same thing in `plotly`.

Having just made the `ggplot` chart, my first instinct was that the syntax:

`geom_image(aes(x = mean_rating, y = win_pct, image=logos))`

might translate directly into something along the lines of:

`add_markers(x = ~mean_rating, y = ~win_pct, marker = ~logo)`

Nope. Plotly uses lists a lot, so then I thought of something along the lines of `marker = list(??? = logo)`. Also nope. According to [the docs](https://plotly.com/r/images/), it turns out you need to add images to `layout()`. Initially, I didn't go this route because I was thinking "surely it can't be that complicated". Unfortunately, it is. Or if there's an easier way, I haven't found it.

### Gather and Wrangle the Data

I'm not going to comment here on the code below because that's not the point of this post. See the comments embedded in the script to get the general idea. I end up with a dataframe `games_long` with 1698 rows and print the first few rows.


```r
library(tidyjson)
library(dplyr)
library(httr)
library(tidyr)
library(ggplot2)
library(ggimage)
library(plotly)

# get vegas betting line data
bet_lines <-
  httr::GET(
    url = paste0("https://api.collegefootballdata.com/lines?year=2021"),
    httr::add_headers(
      Authorization = paste("Bearer", Sys.getenv("YOUR_API_TOKEN"))))

games <- httr::content(bet_lines, "parsed") %>% spread_all

# get data on FBS teams - including links to their logo
fbs <-
  httr::GET(
    url = "https://api.collegefootballdata.com/teams/fbs?year=2021",
    httr::add_headers(
      Authorization = paste("Bearer", Sys.getenv("YOUR_API_TOKEN"))))

fbs_teams <-
  httr::content(fbs, "parsed") %>%
  spread_all

logos <-
  tibble(data = content(fbs, "parsed")) %>%
  unnest_wider(data) %>%
  unnest_wider(logos) %>%
  select(school, `...1`) %>%
  rename("logo" ="...1")

# combine the data sets
games <-
  games %>%
  left_join(
    httr::content(bet_lines, "parsed") %>%
      enter_object(lines) %>%
      gather_array %>%
      spread_all %>%
      mutate(spread = as.numeric(spread)) %>%
      group_by(document.id) %>%
      summarize(home_spread = mean(spread)),
    by = "document.id") %>%
  mutate(home_mov = homeScore - awayScore)

# calculate how each team did against the spread
games <- games %>% mutate(ats = home_spread + home_mov)

# reformat to long data for plotting
games_long <-
  games %>%
  select(homeTeam, home_spread, ats) %>%
  rename("team" = "homeTeam", "spread" = "home_spread") %>%
  bind_rows(
    games %>%
      select(awayTeam, home_spread, ats) %>%
      rename("team" = "awayTeam", "spread" = "home_spread") %>%
      mutate(spread = -spread,
             ats = -ats)) %>%
  filter(team %in% (fbs_teams %>% .$school)) %>%
  group_by(team) %>%
  summarize(mean_spread = mean(spread),
            mean_ats = mean(ats)) %>%
  left_join(logos, by = c("team" = "school"))

head(games_long)
```

```
## # A tibble: 6 x 4
##   team              mean_spread mean_ats logo                                   
##   <chr>                   <dbl>    <dbl> <chr>                                  
## 1 Air Force               -8.47     3.44 http://a.espncdn.com/i/teamlogos/ncaa/~
## 2 Akron                   18.6     -1.11 http://a.espncdn.com/i/teamlogos/ncaa/~
## 3 Alabama                -24.9     -2.57 http://a.espncdn.com/i/teamlogos/ncaa/~
## 4 Appalachian State      -12.4      2.57 http://a.espncdn.com/i/teamlogos/ncaa/~
## 5 Arizona                 11.8     -2.42 http://a.espncdn.com/i/teamlogos/ncaa/~
## 6 Arizona State          -13.4     -4.63 http://a.espncdn.com/i/teamlogos/ncaa/~
```

Here's the plot with `ggplot2`.


```r
games_long %>%
  ggplot() +
  geom_image(aes(x=mean_spread, y=mean_ats, image = logo)) +
  coord_fixed() +
  theme_bw() +
  labs(title = "Team Means",
       x = "Mean Spread",
       y = "Against the Spread")
```

![](/assets/images/plotly-markdown/unnamed-chunk-2-1.png)<!-- -->

To create the plot in `plotly`, I first created `imgs()`, which is a list of lists that will go in the `layout()` function. I'm sure there's a slick way of doing this without a for loop, but this will get the job done.


```r
imgs <- list()

for (i in 1:130){

  l <- list(
    source = games_long[i, 'logo'] %>% .$logo,
    xref = "x",
    yref = "y",
    x = games_long[i, 'mean_spread'] %>% .$mean_spread,
    y = games_long[i, 'mean_ats'] %>% .$mean_ats,
    sizex = 2,
    sizey = 2,
    xanchor = "center",
    yanchor = "middle",
    opacity = 1.0)

  imgs[[i]] <- l
}
```




```r
p <- games_long %>%
  plot_ly() %>%
  add_markers(x=~mean_spread, y=~mean_ats, text=~team) %>%
  layout(
    yaxis = list(
      scaleanchor = "x",
      scaleratio = 1),
    images = imgs
  )

p
```

```{=html}
<div id="htmlwidget-79d7c86f673d6d6e7994" style="width:672px;height:480px;" class="plotly html-widget"></div>
<script type="application/json" data-for="htmlwidget-79d7c86f673d6d6e7994">{"x":{"visdat":{"400cf5a104c":["function () ","plotlyVisDat"]},"cur_data":"400cf5a104c","attrs":{"400cf5a104c":{"alpha_stroke":1,"sizes":[10,100],"spans":[1,20],"x":{},"y":{},"type":"scatter","mode":"markers","text":{},"inherit":true}},"layout":{"margin":{"b":40,"l":60,"t":25,"r":10},"yaxis":{"domain":[0,1],"automargin":true,"scaleanchor":"x","scaleratio":1,"title":"mean_ats"},"images":[{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2005.png","xref":"x","yref":"y","x":-8.47222222222222,"y":3.44444444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2006.png","xref":"x","yref":"y","x":18.5611111111111,"y":-1.10555555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/333.png","xref":"x","yref":"y","x":-24.8769230769231,"y":-2.56923076923077,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2026.png","xref":"x","yref":"y","x":-12.350641025641,"y":2.5724358974359,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/12.png","xref":"x","yref":"y","x":11.8270833333333,"y":-2.42291666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/9.png","xref":"x","yref":"y","x":-13.3777777777778,"y":-4.62777777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/8.png","xref":"x","yref":"y","x":-5.55,"y":1.95,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2032.png","xref":"x","yref":"y","x":9.21111111111111,"y":-4.12222222222222,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/349.png","xref":"x","yref":"y","x":-9.85069444444444,"y":1.39930555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2.png","xref":"x","yref":"y","x":-6.54583333333333,"y":0.870833333333334,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2050.png","xref":"x","yref":"y","x":-0.455555555555556,"y":-2.53888888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/239.png","xref":"x","yref":"y","x":-6.23461538461538,"y":7.07307692307692,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/68.png","xref":"x","yref":"y","x":-6.23125,"y":3.93541666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/103.png","xref":"x","yref":"y","x":-5.4625,"y":-2.9625,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/189.png","xref":"x","yref":"y","x":12.3083333333333,"y":3.05833333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2084.png","xref":"x","yref":"y","x":-2.87847222222222,"y":-3.54513888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/252.png","xref":"x","yref":"y","x":-8.85277777777778,"y":0.397222222222222,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/25.png","xref":"x","yref":"y","x":-1.35486111111111,"y":0.145138888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2117.png","xref":"x","yref":"y","x":-2.37777777777778,"y":4.45555555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2429.png","xref":"x","yref":"y","x":3.40208333333333,"y":-3.43125,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2132.png","xref":"x","yref":"y","x":-19.5807692307692,"y":3.57307692307692,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/228.png","xref":"x","yref":"y","x":-15.3006944444444,"y":-3.46736111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/324.png","xref":"x","yref":"y","x":-21.1173611111111,"y":-0.700694444444445,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/38.png","xref":"x","yref":"y","x":7.22152777777778,"y":-0.695138888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/36.png","xref":"x","yref":"y","x":1.00555555555556,"y":-3.57777777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/41.png","xref":"x","yref":"y","x":21.4409722222222,"y":-1.47569444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/150.png","xref":"x","yref":"y","x":6.89027777777778,"y":-10.0263888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/151.png","xref":"x","yref":"y","x":1.17847222222222,"y":4.59513888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2199.png","xref":"x","yref":"y","x":-1.58541666666667,"y":1.58125,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/57.png","xref":"x","yref":"y","x":-14.3458333333333,"y":-9.09583333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2226.png","xref":"x","yref":"y","x":-2.7125,"y":-3.04583333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2229.png","xref":"x","yref":"y","x":7.125,"y":-12.2083333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/52.png","xref":"x","yref":"y","x":-1.44444444444444,"y":-0.361111111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/278.png","xref":"x","yref":"y","x":-9.52638888888889,"y":3.80694444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/61.png","xref":"x","yref":"y","x":-23.7134615384615,"y":6.13269230769231,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/290.png","xref":"x","yref":"y","x":7.6375,"y":-3.52916666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2247.png","xref":"x","yref":"y","x":2.3625,"y":0.945833333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/59.png","xref":"x","yref":"y","x":5.79375,"y":-3.70625,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/62.png","xref":"x","yref":"y","x":1.76538461538462,"y":-0.85,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/248.png","xref":"x","yref":"y","x":-12.0205128205128,"y":4.28717948717949,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/356.png","xref":"x","yref":"y","x":6.50416666666667,"y":4.75416666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/84.png","xref":"x","yref":"y","x":3.82291666666667,"y":-12.1770833333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2294.png","xref":"x","yref":"y","x":-6.10961538461538,"y":-1.34038461538462,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/66.png","xref":"x","yref":"y","x":-13.1701388888889,"y":-0.920138888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2305.png","xref":"x","yref":"y","x":21.71875,"y":0.302083333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2306.png","xref":"x","yref":"y","x":-2.33958333333333,"y":2.91041666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2309.png","xref":"x","yref":"y","x":0.448717948717949,"y":-2.01282051282051,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/96.png","xref":"x","yref":"y","x":-8.30694444444445,"y":2.94305555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2335.png","xref":"x","yref":"y","x":-14.81875,"y":-4.73541666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/309.png","xref":"x","yref":"y","x":null,"y":null,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2433.png","xref":"x","yref":"y","x":18.6229166666667,"y":6.03958333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2348.png","xref":"x","yref":"y","x":0.922916666666667,"y":-5.07708333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/97.png","xref":"x","yref":"y","x":-2.47916666666667,"y":2.4375,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/99.png","xref":"x","yref":"y","x":-2.62708333333333,"y":-0.877083333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/276.png","xref":"x","yref":"y","x":-11.2125,"y":0.0374999999999998,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/120.png","xref":"x","yref":"y","x":-0.138194444444445,"y":-5.30486111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/235.png","xref":"x","yref":"y","x":-3.59652777777778,"y":-2.76319444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2390.png","xref":"x","yref":"y","x":-5.36597222222222,"y":0.300694444444445,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/193.png","xref":"x","yref":"y","x":-3.25416666666667,"y":1.99583333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/130.png","xref":"x","yref":"y","x":-10.5423076923077,"y":11.0730769230769,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/127.png","xref":"x","yref":"y","x":-2.21736111111111,"y":4.03263888888889,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2393.png","xref":"x","yref":"y","x":3.15208333333333,"y":7.06875,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/135.png","xref":"x","yref":"y","x":-4.07291666666667,"y":3.76041666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/344.png","xref":"x","yref":"y","x":-4.5875,"y":1.07916666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/142.png","xref":"x","yref":"y","x":-0.327083333333333,"y":-5.32708333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2426.png","xref":"x","yref":"y","x":9.66041666666667,"y":1.49375,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/152.png","xref":"x","yref":"y","x":-7.3,"y":6.11666666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/158.png","xref":"x","yref":"y","x":-2.80694444444444,"y":2.44305555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2440.png","xref":"x","yref":"y","x":-8.12430555555556,"y":4.12569444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/167.png","xref":"x","yref":"y","x":9.41875,"y":-6.83125,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/166.png","xref":"x","yref":"y","x":20.3548611111111,"y":2.52152777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/153.png","xref":"x","yref":"y","x":-10.1416666666667,"y":-5.30833333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/249.png","xref":"x","yref":"y","x":5.82152777777778,"y":6.90486111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2459.png","xref":"x","yref":"y","x":4.35705128205128,"y":3.20320512820513,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/77.png","xref":"x","yref":"y","x":4.34583333333333,"y":-8.07083333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/87.png","xref":"x","yref":"y","x":-8.43958333333333,"y":8.56041666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/195.png","xref":"x","yref":"y","x":2.00902777777778,"y":-5.74097222222222,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/194.png","xref":"x","yref":"y","x":-19.7895833333333,"y":4.79375,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/201.png","xref":"x","yref":"y","x":-17.7027777777778,"y":-4.53611111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/197.png","xref":"x","yref":"y","x":-8.46153846153846,"y":5.38461538461539,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/295.png","xref":"x","yref":"y","x":7.79236111111111,"y":8.70902777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/145.png","xref":"x","yref":"y","x":-8.06041666666667,"y":2.85625,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2483.png","xref":"x","yref":"y","x":-11.0044871794872,"y":-5.08141025641026,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/204.png","xref":"x","yref":"y","x":-3.36458333333333,"y":3.46875,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/213.png","xref":"x","yref":"y","x":-7.73055555555556,"y":1.76944444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/221.png","xref":"x","yref":"y","x":-12.6096153846154,"y":7.31346153846154,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2509.png","xref":"x","yref":"y","x":-2.63541666666667,"y":4.36458333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/242.png","xref":"x","yref":"y","x":7.70069444444444,"y":-6.96597222222222,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/164.png","xref":"x","yref":"y","x":2.53888888888889,"y":-1.54444444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/21.png","xref":"x","yref":"y","x":-7.15769230769231,"y":-0.0807692307692307,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/23.png","xref":"x","yref":"y","x":-2.18055555555556,"y":-8.68055555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2567.png","xref":"x","yref":"y","x":-9.44166666666667,"y":0.558333333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/6.png","xref":"x","yref":"y","x":1.10763888888889,"y":-0.392361111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2579.png","xref":"x","yref":"y","x":3.76388888888889,"y":0.847222222222222,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/58.png","xref":"x","yref":"y","x":12.1770833333333,"y":0.677083333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2572.png","xref":"x","yref":"y","x":8.80902777777778,"y":-1.44097222222222,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/24.png","xref":"x","yref":"y","x":6.55,"y":-5.45,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/183.png","xref":"x","yref":"y","x":3.15069444444444,"y":1.73402777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2628.png","xref":"x","yref":"y","x":-3.06944444444444,"y":-9.31944444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/218.png","xref":"x","yref":"y","x":9.73541666666667,"y":-11.43125,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2633.png","xref":"x","yref":"y","x":-6.38472222222222,"y":4.94861111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/251.png","xref":"x","yref":"y","x":-6.34791666666667,"y":-2.18125,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/245.png","xref":"x","yref":"y","x":-12.8416666666667,"y":0.575,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/326.png","xref":"x","yref":"y","x":6.35763888888889,"y":-3.55902777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2641.png","xref":"x","yref":"y","x":0.553472222222222,"y":-1.52986111111111,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2649.png","xref":"x","yref":"y","x":-11.7729166666667,"y":1.39375,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2653.png","xref":"x","yref":"y","x":-2.19791666666667,"y":-5.44791666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2655.png","xref":"x","yref":"y","x":4.74236111111111,"y":-1.67430555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/202.png","xref":"x","yref":"y","x":-1.68333333333333,"y":-3.01666666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/5.png","xref":"x","yref":"y","x":-5.78055555555556,"y":0.802777777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2116.png","xref":"x","yref":"y","x":-10.7291666666667,"y":-3.72916666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/26.png","xref":"x","yref":"y","x":-5.94166666666667,"y":3.80833333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/113.png","xref":"x","yref":"y","x":23.4569444444444,"y":-3.29305555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2439.png","xref":"x","yref":"y","x":15.50625,"y":3.42291666666667,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/30.png","xref":"x","yref":"y","x":-4.13125,"y":-7.21458333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2636.png","xref":"x","yref":"y","x":-10.5262820512821,"y":3.70448717948718,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/254.png","xref":"x","yref":"y","x":-10.125,"y":4.72115384615385,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/328.png","xref":"x","yref":"y","x":-0.0807692307692307,"y":7.84230769230769,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2638.png","xref":"x","yref":"y","x":2.02083333333333,"y":2.4375,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/238.png","xref":"x","yref":"y","x":16.9583333333333,"y":-3.125,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/258.png","xref":"x","yref":"y","x":-2.58055555555556,"y":0.169444444444444,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/259.png","xref":"x","yref":"y","x":-2.98541666666667,"y":-1.06875,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/154.png","xref":"x","yref":"y","x":-8.22692307692308,"y":2.69615384615385,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/264.png","xref":"x","yref":"y","x":-3.95763888888889,"y":-5.12430555555556,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/265.png","xref":"x","yref":"y","x":0.216666666666667,"y":4.38333333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/277.png","xref":"x","yref":"y","x":-3.26875,"y":-0.76875,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/98.png","xref":"x","yref":"y","x":-7.75192307692308,"y":6.63269230769231,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2711.png","xref":"x","yref":"y","x":-4.85277777777778,"y":-2.60277777777778,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/275.png","xref":"x","yref":"y","x":-10.6625,"y":-1.24583333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1},{"source":"http://a.espncdn.com/i/teamlogos/ncaa/500/2751.png","xref":"x","yref":"y","x":-5.1875,"y":-4.52083333333333,"sizex":2,"sizey":2,"xanchor":"center","yanchor":"middle","opacity":1}],"xaxis":{"domain":[0,1],"automargin":true,"title":"mean_spread"},"hovermode":"closest","showlegend":false},"source":"A","config":{"modeBarButtonsToAdd":["hoverclosest","hovercompare"],"showSendToCloud":false},"data":[{"x":[-8.47222222222222,18.5611111111111,-24.8769230769231,-12.350641025641,11.8270833333333,-13.3777777777778,-5.55,9.21111111111111,-9.85069444444444,-6.54583333333333,-0.455555555555556,-6.23461538461538,-6.23125,-5.4625,12.3083333333333,-2.87847222222222,-8.85277777777778,-1.35486111111111,-2.37777777777778,3.40208333333333,-19.5807692307692,-15.3006944444444,-21.1173611111111,7.22152777777778,1.00555555555556,21.4409722222222,6.89027777777778,1.17847222222222,-1.58541666666667,-14.3458333333333,-2.7125,7.125,-1.44444444444444,-9.52638888888889,-23.7134615384615,7.6375,2.3625,5.79375,1.76538461538462,-12.0205128205128,6.50416666666667,3.82291666666667,-6.10961538461538,-13.1701388888889,21.71875,-2.33958333333333,0.448717948717949,-8.30694444444445,-14.81875,18.6229166666667,0.922916666666667,-2.47916666666667,-2.62708333333333,-11.2125,-0.138194444444445,-3.59652777777778,-5.36597222222222,-3.25416666666667,-10.5423076923077,-2.21736111111111,3.15208333333333,-4.07291666666667,-4.5875,-0.327083333333333,9.66041666666667,-7.3,-2.80694444444444,-8.12430555555556,9.41875,20.3548611111111,-10.1416666666667,5.82152777777778,4.35705128205128,4.34583333333333,-8.43958333333333,2.00902777777778,-19.7895833333333,-17.7027777777778,-8.46153846153846,7.79236111111111,-8.06041666666667,-11.0044871794872,-3.36458333333333,-7.73055555555556,-12.6096153846154,-2.63541666666667,7.70069444444444,2.53888888888889,-7.15769230769231,-2.18055555555556,-9.44166666666667,1.10763888888889,3.76388888888889,12.1770833333333,8.80902777777778,6.55,3.15069444444444,-3.06944444444444,9.73541666666667,-6.38472222222222,-6.34791666666667,-12.8416666666667,6.35763888888889,0.553472222222222,-11.7729166666667,-2.19791666666667,4.74236111111111,-1.68333333333333,-5.78055555555556,-10.7291666666667,-5.94166666666667,23.4569444444444,15.50625,-4.13125,-10.5262820512821,-10.125,-0.0807692307692307,2.02083333333333,16.9583333333333,-2.58055555555556,-2.98541666666667,-8.22692307692308,-3.95763888888889,0.216666666666667,-3.26875,-7.75192307692308,-4.85277777777778,-10.6625,-5.1875],"y":[3.44444444444444,-1.10555555555556,-2.56923076923077,2.5724358974359,-2.42291666666667,-4.62777777777778,1.95,-4.12222222222222,1.39930555555556,0.870833333333334,-2.53888888888889,7.07307692307692,3.93541666666667,-2.9625,3.05833333333333,-3.54513888888889,0.397222222222222,0.145138888888889,4.45555555555556,-3.43125,3.57307692307692,-3.46736111111111,-0.700694444444445,-0.695138888888889,-3.57777777777778,-1.47569444444444,-10.0263888888889,4.59513888888889,1.58125,-9.09583333333333,-3.04583333333333,-12.2083333333333,-0.361111111111111,3.80694444444444,6.13269230769231,-3.52916666666667,0.945833333333333,-3.70625,-0.85,4.28717948717949,4.75416666666667,-12.1770833333333,-1.34038461538462,-0.920138888888889,0.302083333333333,2.91041666666667,-2.01282051282051,2.94305555555556,-4.73541666666667,6.03958333333333,-5.07708333333333,2.4375,-0.877083333333333,0.0374999999999998,-5.30486111111111,-2.76319444444444,0.300694444444445,1.99583333333333,11.0730769230769,4.03263888888889,7.06875,3.76041666666667,1.07916666666667,-5.32708333333333,1.49375,6.11666666666667,2.44305555555556,4.12569444444444,-6.83125,2.52152777777778,-5.30833333333333,6.90486111111111,3.20320512820513,-8.07083333333333,8.56041666666667,-5.74097222222222,4.79375,-4.53611111111111,5.38461538461539,8.70902777777778,2.85625,-5.08141025641026,3.46875,1.76944444444444,7.31346153846154,4.36458333333333,-6.96597222222222,-1.54444444444444,-0.0807692307692307,-8.68055555555556,0.558333333333333,-0.392361111111111,0.847222222222222,0.677083333333333,-1.44097222222222,-5.45,1.73402777777778,-9.31944444444444,-11.43125,4.94861111111111,-2.18125,0.575,-3.55902777777778,-1.52986111111111,1.39375,-5.44791666666667,-1.67430555555556,-3.01666666666667,0.802777777777778,-3.72916666666667,3.80833333333333,-3.29305555555556,3.42291666666667,-7.21458333333333,3.70448717948718,4.72115384615385,7.84230769230769,2.4375,-3.125,0.169444444444444,-1.06875,2.69615384615385,-5.12430555555556,4.38333333333333,-0.76875,6.63269230769231,-2.60277777777778,-1.24583333333333,-4.52083333333333],"type":"scatter","mode":"markers","text":["Air Force","Akron","Alabama","Appalachian State","Arizona","Arizona State","Arkansas","Arkansas State","Army","Auburn","Ball State","Baylor","Boise State","Boston College","Bowling Green","Buffalo","BYU","California","Central Michigan","Charlotte","Cincinnati","Clemson","Coastal Carolina","Colorado","Colorado State","Connecticut","Duke","East Carolina","Eastern Michigan","Florida","Florida Atlantic","Florida International","Florida State","Fresno State","Georgia","Georgia Southern","Georgia State","Georgia Tech","Hawai'i","Houston","Illinois","Indiana","Iowa","Iowa State","Kansas","Kansas State","Kent State","Kentucky","Liberty","Louisiana Monroe","Louisiana Tech","Louisville","LSU","Marshall","Maryland","Memphis","Miami","Miami (OH)","Michigan","Michigan State","Middle Tennessee","Minnesota","Mississippi State","Missouri","Navy","NC State","Nebraska","Nevada","New Mexico","New Mexico State","North Carolina","North Texas","Northern Illinois","Northwestern","Notre Dame","Ohio","Ohio State","Oklahoma","Oklahoma State","Old Dominion","Ole Miss","Oregon","Oregon State","Penn State","Pittsburgh","Purdue","Rice","Rutgers","San Diego State","San José State","SMU","South Alabama","South Carolina","South Florida","Southern Mississippi","Stanford","Syracuse","TCU","Temple","Tennessee","Texas","Texas A&M","Texas State","Texas Tech","Toledo","Troy","Tulane","Tulsa","UAB","UCF","UCLA","UMass","UNLV","USC","UT San Antonio","Utah","Utah State","UTEP","Vanderbilt","Virginia","Virginia Tech","Wake Forest","Washington","Washington State","West Virginia","Western Kentucky","Western Michigan","Wisconsin","Wyoming"],"marker":{"color":"rgba(31,119,180,1)","line":{"color":"rgba(31,119,180,1)"}},"error_y":{"color":"rgba(31,119,180,1)"},"error_x":{"color":"rgba(31,119,180,1)"},"line":{"color":"rgba(31,119,180,1)"},"xaxis":"x","yaxis":"y","frame":null}],"highlight":{"on":"plotly_click","persistent":false,"dynamic":false,"selectize":false,"opacityDim":0.2,"selected":{"opacity":1},"debounce":0},"shinyEvents":["plotly_hover","plotly_click","plotly_selected","plotly_relayout","plotly_brushed","plotly_brushing","plotly_clickannotation","plotly_doubleclick","plotly_deselect","plotly_afterplot","plotly_sunburstclick"],"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script>
```

... and if this was an RMarkdown script, you'd see a nice `plotly` version. However, I'm using GitHub Pages to create a personal website and make these posts, which renders `.md` files but not `.Rmd` files. This brings me to the second goal of this post.

## How To Display Plotly Graphs On GitHub Pages

After a lot of Googling and trial and error, this finally worked for me. First, save the plot as an `htmlwidget`. The `libdir` is the path in my GitHub repo for images and other related things for this particular post.


```r
htmlwidgets::saveWidget(
  widget = p,
  file = "p1.html",
  selfcontained = F,
  libdir = "/assets/images/plotly-markdown/lib")
```

Finally, in a markdown portion of the file (not a code chunk), I can use this to render the plot:

```
<iframe src="/assets/images/plotly-markdown/p1.html" width="100%" height="800" id="igraph" scrolling="no" seamless="seamless" frameBorder="0"> </iframe>
```
<iframe src="/assets/images/plotly-markdown/p1.html" width="100%" height="800" id="igraph" scrolling="no" seamless="seamless" frameBorder="0"> </iframe>
