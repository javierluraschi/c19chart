Coronavirus Deaths by Region per Capita
================

``` r
library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
library(magrittr)

load(pins::pin("https://github.com/RamiKrispin/coronavirus/raw/master/data/coronavirus.rda"))

coronavirus <- coronavirus %>%
  mutate(country = `Country.Region`) %>%
  mutate(
    country = case_when(
      country == "United Kingdom" ~ "UK",
      country == "Korea, South" ~ "South Korea",
      TRUE ~ country
    )
  )

coronavirus <- coronavirus %>%
  filter(type == "death") %>%
  group_by(country, date) %>%
  summarise(cases = sum(cases))
```

``` r
since_start <- coronavirus %>%
  group_by(country) %>%
  mutate(total = cumsum(cases)) %>%
  filter(total >= 10) %>%
  group_by(country) %>%
  mutate(since_start = 1:n())
```

``` r
library(ggplot2)
library(directlabels)

# Validate against current chart https://twitter.com/harryrutter/status/1244253749520084992/photo/1
since_start %>%
  ggplot(aes(x = since_start, y = total, color = country)) +
    geom_point(size = 0.5) +
    geom_line(alpha = 0.3) +
    scale_colour_discrete(guide = 'none') +
    scale_x_discrete(expand = c(0, 8)) +
    scale_y_continuous(trans='log2') +
    geom_dl(aes(label = country), method = list(dl.trans(x = x + 0.2), "last.points", cex = 0.8)) +
    ggtitle(label = "Country by Country: How COVID-19 case trajectories compare",
            subtitle = "Cumulative number of confirmed cases, by number of days since 10th death") +
    theme_light()
```

![](README_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

``` r
population <- read.csv(pins::pin("https://datahub.io/JohnSnowLabs/population-figures-by-country/r/population-figures-by-country-csv.csv"), stringsAsFactors = F)

population <- population %>%
  mutate(
    country = case_when(
      Country == "United States" ~ "US",
      Country == "United Kingdom" ~ "UK",
      Country == "Czech Republic" ~ "Czechia",
      Country == "Iran, Islamic Rep." ~ "Iran",
      Country == "Korea, Rep." ~ "South Korea",
      Country == "Russian Federation" ~ "Russia",
      Country == "Brunei Darussalam" ~ "Brunei",
      Country == "Congo, Rep." ~ "Congo (Kinshasa)",
      Country == "Egypt, Arab Rep." ~ "Egypt",
      Country == "Kyrgyz Republic" ~ "Kyrgyzstan",
      Country == "Macedonia, FYR" ~ "North Macedonia",
      Country == "Slovak Republic" ~ "Slovakia",
      Country == "Venezuela, RB" ~ "Venezuela",
      Country == "Bahamas, The" ~ "Bahamas",
      Country == "Myanmar"  ~ "Burma",
      Country == "Gambia, The" ~ "Gambia",
      Country == "Syrian Arab Republic" ~ "Syria",
      Country == "Congo, Dem. Rep." ~ "Congo (Brazzaville)",
      TRUE ~ Country
    )
  )

population <- transmute(population, country = country, population = Year_2016)

population <- rbind(population, data.frame(
  country = c("Taiwan*", "Diamond Princess", "MS Zaandam"),
  population = c(23780000, 3700, 136 + 97)
))

countries <- unique(since_start$country)
countries_missing <- countries[!countries %in% unique(population$country)]
if (length(countries_missing) > 0) stop("Countries missing in population table: ", paste0(countries_missing, collapse = "; "))
```

``` r
covid_since_start <- since_start %>%
  left_join(population, by = "country") %>%
  mutate(total_percent = total / population)

countries_highlight <- c("San Marino", "Andorra", "Iceland", "India", "Luxembourg", "US", "Spain", "Italy", "Japan", "China", "Iran", "UK", "South Korea", "Singapore", "Diamond Princess", "Vietnam", "Taiwan*", "Germany", "France", "Monaco", "Sweden", "Philippines", "Belgium", "Brazil", "Taiwan*", "Mexico")

highlight_since_start <- filter(covid_since_start, country %in% countries_highlight)

population_us <- population %>% filter(country == "US") %>% pull(population)
total_us <- coronavirus %>% 
  filter(country == "US") %>%
  summarise(total = sum(cases)) %>%
  pull(total)

# https://www.cdc.gov/flu/about/burden/2018-2019.html
deaths_influenza <- data.frame(since_start = 1:120, total_percent = 1:120 * (34200 / 365) / population_us)
# https://www.cdc.gov/nchs/fastats/leading-causes-of-death.htm
deaths_clrd <- data.frame(since_start = 1:120, total_percent = 1:120 * (160201 / 365) / population_us)
# https://www.cdc.gov/heartdisease/facts.htm
deaths_heart <- data.frame( since_start = 1:120, total_percent = 1:120 * (647000 / 365) / population_us)

highlight_since_start %>%
  ggplot(aes(x = since_start, y = total_percent, colour = country)) +
    geom_line(aes(group = country), data = covid_since_start, alpha = 0.3, colour = "grey") +
    geom_line(alpha = 0.8) +
    scale_colour_discrete(guide = 'none') +
    scale_x_continuous(limits = c(0, 95)) +
    scale_y_continuous(labels = scales::percent_format(accuracy = 0.00001), trans='log2') +
    geom_dl(aes(label = country), method = list("last.bumpup", cex = 0.7)) +
    xlab("Number of days since tenth death") + ylab("") +
    labs(title = "Coronavirus Deaths by Region per Capita",
         subtitle = "Cumulative number of deaths as percent of population since tenth death",
         caption = element_text("Source: Johns Hopkins University Center for Systems Science and Engineering. Data Updated: April 25, 2020\nFigure: github.com/javierluraschi/c19chart", color="grey")) +
    theme_bw() + 
    theme(plot.background = element_rect(fill = "#fff1e6"),
          panel.background = element_rect(fill = "#fff1e6",
                                colour = "lightblue",
                                size = 0.5, linetype = "solid"),
          plot.caption = element_text(color="grey60"),
          axis.line.x = element_line(colour = "black"),
          panel.grid.major = element_line(color="grey90"),
          panel.grid.minor = element_line(color="grey90"),
          panel.border = element_blank()) +
    geom_line(data = deaths_influenza, alpha = 0.3, colour="grey70", linetype="dashed") +
    annotate("text", x=87.2, y = max(deaths_influenza$total_percent) + 0.000011, label = "influenza (93/day US)", size = 3, colour="grey70") +
    geom_line(data = deaths_clrd, alpha = 0.3, colour="grey70", linetype="dashed") +
    annotate("text", x=78.0, y = max(deaths_clrd$total_percent) + 0.00007, label = "chronic lower respiratory diseases (438/day US)", size = 3, colour="grey70") +
    geom_line(data = deaths_heart, alpha = 0.3, colour="grey70", linetype="dashed") +
    annotate("text", x=84.7, y = max(deaths_heart$total_percent) + 0.00026, label = "heart disease (1772/day US)", size = 3, colour="grey70") +
    ggsave("covid19-deaths-per-capita.png", device = "png", width = 10, height = 5)
```

    ## Warning: Removed 25 row(s) containing missing values (geom_path).
    
    ## Warning: Removed 25 row(s) containing missing values (geom_path).
    
    ## Warning: Removed 25 row(s) containing missing values (geom_path).
    
    ## Warning: Removed 25 row(s) containing missing values (geom_path).
    
    ## Warning: Removed 25 row(s) containing missing values (geom_path).
    
    ## Warning: Removed 25 row(s) containing missing values (geom_path).

![](README_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

``` r
covid_since_start %>%
  filter(since_start <= 21) %>%
  group_by(country) %>%
  summarise(total = sum(cases), days = max(since_start)) %>%
  left_join(population, by = "country") %>%
  ggplot(aes(x=population, y=total)) +
  scale_x_continuous(trans='log2') +
  scale_y_continuous(trans='log2') +
  geom_dl(aes(label = country), method = list("last.bumpup", cex = 0.7, hjust=-0.2)) +
  geom_smooth(method=lm) +
  labs(title = "Countries deaths vs population after 10th death on 21th day") +
  ggsave("covid19-deaths-population.png", device = "png", width = 12, height = 6)
```

    ## `geom_smooth()` using formula 'y ~ x'
    ## `geom_smooth()` using formula 'y ~ x'

![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->
