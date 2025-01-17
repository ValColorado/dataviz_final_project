---
title: "Mini-Project 02"
output: 
  html_document:
    keep_md: true
    toc: true
    toc_float: true
---

# Data Visualization Project 02: Finishers of the Boston Marathon of 2017

**Here we go!** 🏃💨

![](images/race.gif)

# Data Wrangling

``` r
# Load required libraries
library(ggplot2)
library(tidyverse)
library(ggthemes)
library(plotly)
```

``` r
# Read the CSV file
race <- read_csv("https://raw.githubusercontent.com/reisanar/datasets/master/marathon_results_2017.csv")
```


For this analysis I am focusing on US racers so I need to filter out the other countries
``` r
 race <- race %>% 
  filter(Country =="USA")  %>% 
  na.omit()
```

Now that its racers from the states I need to filter out american territories 
``` r
race <- subset(race, !(State %in% c("AA", "AE","AP","MH","GU","PR","VI","DC")))
```

To make it easier to manipulate the columns I added a "_" where there was white space and I renamed "M/F" to genders
``` r
colnames(race) <- gsub("\\s", "_", colnames(race))
```

``` r
race <- rename(race, genders = `M/F`)
```

now that I have the racers I need to calculate the average run time in hours 
``` r
avg_time <-race %>% 
  group_by(Age) %>% 
  summarise(avg_overall_time = mean(as.numeric(Official_Time), na.rm = TRUE),
            avg_overall_time = avg_overall_time/3600) 

avg_time
```


## Interactive Plot

Here is where I group by age and find the average overall time for each age group.

``` r
avg_time_plot<-
  ggplot(data = avg_time) +
  geom_point(aes(x = Age, y = avg_overall_time,text = row.names(avg_overall_time))) +
  scale_x_continuous(limits = c(18, NA), breaks = seq(18, max(avg_time$Age), by = 5)) +
  labs(x = "Age", y = "Average Finish Time In Hours", title = "Average Finish Time Per Age Group For Runners In USA")+  
  theme(axis.text.x = element_text(angle = 25, hjust = 1))+
  theme_light()
```

``` r
avg_time_plot <- ggplotly(avg_time_plot)
```


```{r}
#htmlwidgets::saveWidget(avg_time_plot, "fancy_plot_avg_time_plot.html")
```

Interactive plots are so powerful. Hovering over each data point shows us more information on each observation! In this example when you hover over you can see that the runner is 48 with an average run time of about 3.9 hours.All of these interactive graphs were saved in `../figures/html plots`

![](../figures/final.jpg)

Here we can see that 28-38 year old typically have the quickest marathon race times. It was interesting to see the 84 year old finisher! I can't imagine running for 6 hours a 25 year old.

After seeing this breakdown I wanted to see how different the average finish times were for males and females

``` r
avg_time_gender <- race %>% 
  group_by(Age,genders) %>% 
  summarise(avg_overall_time = mean(as.numeric(Official_Time), na.rm = TRUE),
            avg_overall_time = avg_overall_time/3600,
)
```

``` r
avg_time_gender_plot <- ggplot(data = avg_time_gender) +
  geom_point(aes(x = Age, y = avg_overall_time,fill=genders)) +
  facet_wrap(~ genders, ncol = 2) +  # Adjust the ncol parameter
  scale_x_continuous(limits = c(18, NA), breaks = seq(18, max(avg_time$Age), by = 5)) +
  labs(label = FALSE,x = "Age", y = "Finish Time In Hours", title = "Average Finish Time For Female and Male Runners In USA") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1), 
        axis.title.x = element_text(margin = margin(t = 10, unit = "pt")),legend.position = "none")+
  theme_light()+
    scale_fill_manual(values = c("#FF00FF", "#4169E1"))  # Specify your desired fill colors
```

``` r
ggplotly(avg_time_gender_plot)%>%
  layout(width = 950)  # Adjust the width value
```

``` r
#ggsave("plot_gender_new.jpg", plot = avg_time_gender_plot, width = 15, height = 8, dpi = 300, type = "cairo")
```

![](../figures/plot_gender_new.jpg)

This graph better visualizes the average finish time for males and females. It was interesting to see the majority of men finish under 4 hours between the age of 18-53 while most females average just over 4 hours

## Spatial Visualization

``` r
library(sf)
```

``` r
# Load world shapefile from Natural Earth
# https://www.naturalearthdata.com/downloads/110m-cultural-vectors/
world_shapes <- read_sf("spatial-data/ne_50m_admin_1_states_provinces/ne_50m_admin_1_states_provinces.shp")
```

Since `world_shapes` and `race` both have state names abbreviated i will rename state to match postal so i am able to join the tables

``` r
race <- rename(race, postal = `State`)
```

There are other countries listed I just want to focus on the avg run time per state in the USA

``` r
avg_state <- race %>% 
  filter(Country =="USA")  %>% 
  group_by(postal) %>% 
  summarise(avg_overall_time = mean(as.numeric(Official_Time), na.rm = TRUE),
            avg_overall_time = avg_overall_time/3600,
) %>% 
  na.omit()
```

``` r
avg_state %>% 
  arrange()
```

Leaving out american territories and military bases

``` r
avg_state <- subset(avg_state, !(postal %in% c("AA", "AE","AP","MH","GU","PR","VI","DC","AK")))
```

Changing the shapes dataset to only have USA

``` r
usa_shapes <- world_shapes %>% 
  filter(admin =="United States of America")
```

``` r
#join the two 
usa_shapes<-usa_shapes %>% 
  left_join(avg_state, by="postal")
```

``` r
usa_shapes <- usa_shapes %>% 
  select(postal,avg_overall_time)  %>% 
  na.omit()
```

This map shows the average finish time for each state. It showed Alaska as the state with the quickest time.

``` r
map_interactive <- ggplot() +
  geom_sf(data = usa_shapes, aes(fill = avg_overall_time, text = paste("State: ", postal)),
          color = "white") +
  scale_fill_gradient(low = "#66a0ff", high = "#4b595e") +
  labs(title = "Average Finish Time For Each State", fill = "Finish Time In Hours", caption = "Finishers of the Boston Marathon of 2017") +
  theme(legend.position = "bottom") +
  theme_void() 
```

``` r
ggplotly(map_interactive)
```


![](../figures/map.jpg)

This map shows how the average finishing time for each state. I wanted an interactive map so when you hovered over each state you know what state it is plus that's specific's state average finish time. This dataset was from the Boston Marathon I was kind of surprised Massachusetts didn't have a quicker finish time. It was interesting to see how people from Alaska had the quickest average.

## Model 

``` r
race_usa <- race %>% 
  filter(Country =="USA")  %>% 
  group_by(postal,Age) %>% 
  summarise(avg_overall_time = mean(as.numeric(Official_Time), na.rm = TRUE),
            avg_overall_time = avg_overall_time/3600,
) %>% 
  na.omit()
```

``` r
race_usa <- subset(race_usa, !(postal %in% c("AA", "AE","AP","MH","GU","PR","VI","DC")))
race_usa
```

``` r
race_model <-ggplot(race_usa, aes(x = Age, y = avg_overall_time)) +
  geom_point(alpha = 0.09) +
  geom_smooth(method = "lm", se = FALSE, color = "#3B8DBD") +
  scale_x_continuous(limits = c(18, NA), breaks = seq(18, max(avg_time$Age), by = 5)) +
  labs(title = "Relationship between Age and Finish Time in Races for American Athleats", y = "Finish Time in Hours", caption = "Finishers of the Boston Marathon of 2017") +
  theme(
    plot.title = element_text(hjust = -.5, family = "Arial", face = "bold", size = 12),
    axis.title.y = element_text(family = "Arial",  size = 12),
    axis.title.x = element_text(family = "Arial", size = 12,margin = margin(t = 10, unit = "pt"))
    )+
theme_light()
```

``` r
race_model
```

![](../figures/race_model.png)

Since my earlier visualizations focused on age and race time, I decided to model the relationship between age and race time for American racers. This shows the race time gradually increases as the racers age increases which is expected. Since I learned more about colors I changed the default blue to the official blue Boston color `#3B8DBD`



We've reached the end of the data visualization! 
![](images/finishline.gif)
