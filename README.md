# Excluding-values-in-multiple-column-mean-calculations
Grabbed row-wise means for multiple columns and excluded multiple values in those means

# The task
I have data that's 1 row per person where people reported the number of drinks they had in a specific time-block. The columns are various time blocks, but the participant could also provide other answers that were coded differently:
* ```NA``` means that they didn't use alcohol at all
* ```0``` mean that they had no drinks at that time
* ```-999``` means that they did drink in that time block, but they can't remember how much. 

Here's the fun part. We want to impute a typical person-mean for ```-999```. In practice, this means that if a person reported they drank in the time block but can't remember how much, we suggest they drank whatever is typical for them. 

However, the mean we want to impute needs to exclude ```-999```, ```0```, and ```NA```.

# Libraries
I will complete this task using the following libraries.
```
# Read in the .csv file
library(readr)
# Chains using the pipe
library(dplyr)
# Other functions
library(purrr)
```
# The data
The data I completed this task with originally had `2,348` variables and about `500` observations. I've created a fake version of this data since the original data was confidential. This fake data is stored [here](fakedata.csv). 

```
#Read in the data
fd.df <- read.csv("fakedata.csv", header = T, sep = ",", na.strings = "NA")

#preview the data
head(fd.df)
```
It's always a good idea to ensure your data is truly 1 row/person. I checked for duplicates by ```ResponseID``` to make sure:
```
fd.df$ResponseId[duplicated(fd.df$ResponseId)]
```
```character(0)``` was returned so there were no duplicates. 

# The function

I'm going to create a function that does the following:
For all columns in the dataframe that are within the right index ```fd.df[2:121]```, by row, take the mean so long as it's greater than ```0``` and not ```NA```. 
To do this, we first need to make sure that everything is read in as numeric. This is done in the second line starting with ```mutate_at()```. Here, all columns that were supposed to be computed in this mean (index `[2:121]`) also start with "AD". Thus, I pull all columns that start with "AD" and make them numeric (```as.numeric()```) and let it recognize that any ```NA``` values are real (```na.rm = T```). I used `starts_with()` to show different functionality. 
```
final.df <- fd.df %>% 
  mutate_at(vars(starts_with("AD")), as.numeric, na.rm = T) %>%
  mutate(avg = apply(.[2:121],1, function(.) mean(.[.>0], na.rm =T)))
```
For the third code line starting with ```mutate()```: The ```(.)``` refers to the whole dataframe, but I'm specifying the index that I want the function to apply here: ```(.[2,121]``` this is required if you have other data in your dataframe, such as ID numbers. The ```1``` refers to row-wise. I could have put a ```2``` for column-wise, if that was my goal. This is an elegant alternative to using ```group_by(ResponseID)``` which is also an option. 

Next, I'm telling ```apply()``` that I want to (apply) use a function and that function is the ```mean()``` of the dataframe ```(.)``` so long as the values in the dataframe are greater than 0 and I want to keep my ```NA``` values (```(.[.>0], na.rm =T ```). Using any value greater than 0 will also exclude my ```-999``` which I want to impute later. 

# Impute the mean

I'm going to add this line of code to the code above: ``` mutate_at(vars(starts_with("AD")), ~if_else(. == -999,  avg, .))```
and it will appear as:
```
final.df <- fd.df %>% 
  mutate_at(vars(starts_with("AD")), as.numeric, na.rm = T) %>%
  mutate(avg = apply(.[2:121],1, function(.) mean(.[.>0], na.rm =T)))%>%
  mutate_at(vars(starts_with("AD")), ~if_else(. == -999,  avg, .))
```
Let's break this down. 
* We are doing the same thing again with ```mutate_at()``` which is just to show functionality. You can do it with indexes too. 
* The reason I have saved a column called `avg` is so I can check to ensure everything is working next. 
* Now my fourth code line reads as follows: for all columns that start with "AD" (aka index 2, 121) apply the following `if_else()` statement to the whole dataframe (denoted with the `~`). 
* The `if_else()` statement reads as follows: if anywhere in the dataframe (denoted by `(.)`) is equal too `-999`, then replace with `avg`. If it's not, keep it the same (denoted by `(.)` again). 

The output will be successfully leaving all other data alone (e.g. ID's, demographics, contact information, etc) and only manipulating the columns specified in `starts_with()` and/or the index you give. 

# Check that it worked
In all of my code, I always encourage checking to make sure your functions/code works the way you are expecting. In this example, I was originally using `rowMeans()` and it was not ignoring values less than 0. I wouldn't have caught that if I forwent this data check. 

For this check, I wanted a column that I knew previously had a value of `-999`. I knew to select the column `AD1B3` using this code:
```
check.df <- fd.df %>% 
  mutate_at(vars(starts_with("AD")), as.numeric, na.rm = T) %>% 
  mutate(avg = apply(.[2:121],1, function(.) mean(.[.>0], na.rm =T))) %>%
  select(2:121, avg) %>% 
  select_if(~min(., na.rm = TRUE) < 0) 
```
When I ran this, the first line had ```-999``` and other values, which was nice.  Otherwise, I probably would have needed to come up with a better way to examine exactly what I wanted (suggestions welcome). 

```
      AD1B3 AD2B3 AD2B4 AD3B3 AD4B3 AD4B4 AD5B3 AD5B4 AD6B3 AD6B4 AD7B3 AD8B3 AD9B2 AD9B3 AD9B4 AD10B3 AD10B4
[1]  -999.0     0     0     0     4  -999     0     0     0     0     0     0     0     0     0      0      0 
```

If I run the same code above on the ```check.df``` for row 1 using ```slice(1)``` and the ```select_if``` to remove all columns that have a value of `0` or `NA` (since neither should be factored into my average), I can evaluate what the ```avg``` column should be:

```
check.df <- fd.df %>% 
  mutate_at(vars(starts_with("AD")), as.numeric, NaN.rm = T, na.rm = T) %>% 
  mutate(avg = apply(.[2:121],1, function(.) mean(.[.>0], na.rm =T))) %>%
  select(2:121, avg) %>% 
  slice(1) %>%
  select_if(~max(., na.rm = TRUE) > 0)  
  ```
```  
   AD4B2  AD4B3 AD12B2 AD12B3 avg
[1]     2     4      4      4  3.5
```
Now, I can see that 3.5 is the correct average. 

I've saved my manipulated data to a dataframe called `final.df`. First, I took a look to make sure the `apply()` function for means was actually excluding all the values I wanted it to. 
```
final.df %>% 
  select(2:121, avg) %>%
  slice(1) %>%
  select_if(~max(., na.rm = TRUE) > 0)
```

This is what will be printed:
```
    AD1B3 AD4B2 AD4B3 AD4B4 AD11B3 AD11B4 AD12B2 AD12B3 AD12B4 avg
[1]   3.5     2     4   3.5    3.5    3.5      4      4    3.5 3.5
```
I know from my ```check.df``` dataframe that `AD1B3` was `-999` and now is replaced with the value in ```avg```. Everything looks good!

# Looking for suggestions
I really want my apply function to work in the `if_else()` statement so once I know it works, I can just run it without saving the `avg` column. Also, I'm new to the `apply()` function so any way to optimize it is also welcomed. 
