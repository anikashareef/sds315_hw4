# sds315_hw4

```{r setup, include=FALSE, message=FALSE}
knitr::opts_chunk$set(echo = TRUE)

library(ggplot2)
library(dplyr)
library(tidyverse)
library(knitr)
library(lubridate)
library(mosaic)
```

*Problem 1*

```{r, message=FALSE, echo=FALSE,fig.width=5, fig.height=3}

#set parameters for simulation 
total_trades <- 2021 
flag_rate <- 0.024 
observed_flags <- 70
n_simulations <- 100000

#run monte carlo simulation 
simulated_flags <- rbinom(n_simulations, total_trades, flag_rate)

#calculate p value: prop of simulations flags greater than or equal to observed trades
flag_pvalue <- mean(simulated_flags >= observed_flags)

#convert to data frame to make a ggplot
simulated_flag_data <- data.frame(simulated_flags)

ggplot(data=simulated_flag_data)+
  geom_histogram(aes(x=simulated_flags), bins=30, color="black",fill='paleturquoise')+
  geom_vline(xintercept=observed_flags, color="hotpink", linetype= "dashed", linewidth=.5)+
  labs( 
    title = "Monte Carlo Simulated Distribution of Flagged Trades",
    x = "Number of Flagged Trades",
    y = "Probability Density")+
  theme_minimal()




```

*problem 2*
```{r, message=FALSE, echo=FALSE,fig.width=6.5, fig.height=3}

#set parameters for simulation
total_inspections <- 50
violation_rate <- 0.03
observed_violations <- 8
n_simulations2 <- 100000

#run monte carlo simulation
simulated_violations <- rbinom(n_simulations2, total_inspections, violation_rate)

#calculated p value: prop of violations greater than or equal to observed violations
violation_pvalue <- mean(simulated_violations >= observed_violations)

#convert to data frame to make a ggplot 
simulated_violations_data <- data.frame(simulated_violations)

ggplot(data=simulated_violations_data)+
  geom_histogram(aes(x=simulated_violations), bins=10, color="black",fill="powderblue")+
  geom_vline(xintercept=observed_violations, color="hotpink", linetype="dashed", linewidth=.5)+
  labs(
    title="Monte Carlo Simulated Distrubution of Health Code Violations",
    x= "Number of Violations",
    y="Probability Density")+
  theme_minimal()


```

*problem 3*

```{r, message=FALSE, echo=FALSE,fig.width=6.5, fig.height=3}

#set the expect distribution for group counts 
expected_distribution = c(G1= 0.30, G2= 0.25, G3= 0.20,G4= 0.15, G5= 0.10)
observed_counts = c(G1= 85, G2= 56, G3= 59, G4= 27, G5= 13)

#total jurors across all trials 
#20 trials x 12 jurors (240)
total_jurors = sum(observed_counts)  

#expected counts under the null 
expected_counts= total_jurors*expected_distribution

#display observed vs expeted counts 
jury_data <- tibble(Group=names(observed_counts), Observed= observed_counts, Expected= expected_counts)
print(jury_data)

#define a function to calculate the chi squared statistic 
chi_squared_jury = function(observed, expected) {
  sum((observed-expected)^2 / expected)
}

#compute chi squared statistic 
observed_chi2 = chi_squared_jury(observed_counts, expected_counts)

#repeat
num_simulations = 10000
chi2= replicate(num_simulations, {
  simulated_jury_counts = as.vector(rmultinom(1, total_jurors, expected_distribution))
  this_chi2 = chi_squared_jury(simulated_jury_counts, expected_counts)
})

#convert to data frame 
chi2_sim_data <- tibble(chi2=chi2)


ggplot(data=chi2_sim_data)+
  geom_histogram(aes(x=chi2), bins=70, color="black",fill="darkcyan")+
  geom_vline(xintercept=observed_chi2, color="hotpink", linetype="dashed", linewidth=.5)+ labs(
  title= "Monte Carlo Simulation of Chi-squared Distribution",
  x= expression(chi^2), 
  y= "Frequency", ) + 
theme_minimal()

#calculate p value by checking the prop of simulated chi sqaure values are greater than the observed one 
p_value_jury= mean(chi2_sim_data$chi2 >= observed_chi2)



# Print summary statistics for the simulated Chi-squared values
print(paste("Observed Chi-squared Statistic:", round(observed_chi2, 4)))
print(paste("Mean of Simulated Chi-squared Values:", round(mean(chi2), 4)))

```

*problem 4*
*part A*
```{r, message=FALSE, echo=FALSE,fig.width=5, fig.height=3}

#load letter frequencies data set
letter_frequencies <- read.csv("letter_frequencies.csv")

# 1 load text file 
#initialize empty vector
brown <- character()

#open file connection
con <- file("brown_sentences.txt", "r")

#read 1000 lines at a time
repeat {
  chunk<- readLines(con, n=1000)
  #stop at end of the file 
  if(length(chunk)==0) break 
  #append 
  brown <- c(brown, chunk)
}
#close connection after reading
close(con)


#2 preprocess text 
#initialize empty list 
letter_counts_list <- list()

#define freq table
freq_table <- data.frame(Letter= LETTERS)

for(i in seq_along(brown)) {
  sentence <- brown[i]
  
  #remove non-letters and convert to uppercase
  clean_sentence = gsub("[^A-Za-z]", "", sentence)
  clean_sentence = toupper(clean_sentence)
  
  #check for empty sentences to remove na values 
  if (nchar(clean_sentence)==0) {
    letter_counts_list[[i]] <- table(factor(character(0), levels= freq_table$Letter))
    next
  }
  
  #count the occurances of each letter in the sentence 
  observed_letter_count= table(factor(strsplit(clean_sentence, "")[[1]], levels=freq_table$Letter))
  
  #store the letter counts in a list
  letter_counts_list[[i]] <- observed_letter_count
}

#3 calculate letter count 
#convert observed letter counts to a data frame 
letter_counts_df <- do.call(rbind, letter_counts_list)

#gof test 
calculate_chi_squared = function(observed_letter_count,letter_frequencies) {
  
  #ensure letter frequencies are normalized and sum to 1 
  letter_frequencies$Probability = letter_frequencies$Probability / sum(letter_frequencies$Probability)
  
  #calculated expected counts 
  total_letters= sum(observed_letter_count)
  
  #if sentences contain no letters return chi squared of 0 
  if (total_letters == 0) return(0)
  
  #calculate expected letter counts 
  expected_letter_count = total_letters*letter_frequencies$Probability
  
  #chi squared stat & account for 0 values 
  chi_squared_letter <- sum((observed_letter_count - expected_letter_count)^2 / 
                             ifelse(expected_letter_count == 0, 1, expected_letter_count))
  
  return(chi_squared_letter)
}

#apply function for each sentence's letter count
chi_squared_values <- apply(letter_counts_df, 1, calculate_chi_squared, letter_frequencies)
#store values in data frame 
chi_squared_df <- data.frame(sentence= seq_along(chi_squared_values), Chi_Squared= chi_squared_values)





#create ggplot 
ggplot(data=chi_squared_df)+
  geom_histogram(aes(x=Chi_Squared), bins=70, color="black", fill= "steelblue1")+
  labs(title = "Distribution of Chi-Squared Values",
       x = expression(chi^2),
       y = "Frequency") +
  theme_minimal()




```

*part B*

```{r, message=FALSE, echo=FALSE,fig.width=5, fig.height=3}

#create vector for the sentences 
sentences <- c(
  "She opened the book and started to read the first chapter, eagerly anticipating what might come next.",
  "Despite the heavy rain, they decided to go for a long walk in the park, crossing the main avenue by the fountain in the center.",
  "The museum’s new exhibit features ancient artifacts from various civilizations around the world.",
  "He carefully examined the document, looking for any clues that might help solve the mystery.",
  "The students gathered in the auditorium to listen to the guest speaker’s inspiring lecture.",
  "Feeling vexed after an arduous and zany day at work, she hoped for a peaceful and quiet evening at home, cozying up after a quick dinner with some TV, or maybe a book on her upcoming visit to Auckland.",
  "The chef demonstrated how to prepare a delicious meal using only locally sourced ingredients, focusing mainly on some excellent dinner recipes from Spain.",
  "They watched the sunset from the hilltop, marveling at the beautiful array of colors in the sky.",
  "The committee reviewed the proposal and provided many points of useful feedback to improve the project’s effectiveness.",
  "Despite the challenges faced during the project, the team worked tirelessly to ensure its successful completion, resulting in a product that exceeded everyone’s expectations."
)

#inatialize empty list 
letter_counts_list2 <- list()

#preprocess 
for(i in seq_along(sentences)) {
  sentence2 <- sentences[i]
  
  #remove non-letters and convert to uppercase
  clean_sentence2 = gsub("[^A-Za-z]", "", sentence2)
  clean_sentence2 = toupper(clean_sentence2)
  
  #check for empty sentences to remove na values 
  if (nchar(clean_sentence2)==0) {
    letter_counts_list2[[i]] <- table(factor(character(0), levels= freq_table$Letter))
    next
  }
  
  #count the occurances of each letter in the sentence 
  observed_letter_count2= table(factor(strsplit(clean_sentence2, "")[[1]], levels=freq_table$Letter))
  
  #store the letter counts in a list
  letter_counts_list2[[i]] <- as.numeric(observed_letter_count2)
}

#3 calculate letter count 
#convert observed letter counts to a data frame 
letter_counts_df2 <- as.data.frame(do.call(rbind, letter_counts_list2))


#gof test 
calculate_chi_squared = function(observed_letter_count2,letter_frequencies) {
  
  #ensure letter frequencies are normalized and sum to 1 
  letter_frequencies$Probability = letter_frequencies$Probability / sum(letter_frequencies$Probability)
  
  #calculated expected counts 
  total_letters= sum(observed_letter_count2)
  
  #if sentences contain no letters return chi squared of 0 
  if (total_letters == 0) return(0)
  
  #calculate expected letter counts 
  expected_letter_count = total_letters*letter_frequencies$Probability
  
  #chi squared stat & account for 0 values 
  chi_squared_values2<- sum((observed_letter_count2 - expected_letter_count)^2 / 
                             ifelse(expected_letter_count == 0, 1, expected_letter_count))
  
  return(chi_squared_values2)
}

#apply functions to comoute observed chi squared value 
observed_chi_squared_10 <- sapply(letter_counts_list2, function(observed_letter_count2){ 
  calculate_chi_squared(observed_letter_count2, letter_frequencies)
  })

#load null distribution
null_distribution <- chi_squared_df$Chi_Squared


#p value function
calculate_p_value <- function(chi_squared_value, null_distribution){
  mean(null_distribution >= chi_squared_value)
}

#calculate p value
p_values <- sapply(observed_chi_squared_10, calculate_p_value, null_distribution)

#p-value table 
p_value_table <- data.frame(
  Sentence = seq_along(sentences),
  Chi_Squared = round(observed_chi_squared_10, 3),
  P_Value = round(p_values, 3)
)

print(p_value_table)

#extract most extreme value 
chi_squared_sentence6 <- p_value_table$Chi_Squared[p_value_table$Sentence == 6]

#create ggplot 
ggplot(data=chi_squared_df)+
  geom_histogram(aes(x=Chi_Squared), bins=70, color="black", fill= "steelblue1")+
  geom_vline(xintercept=chi_squared_sentence6 , color="hotpink", linetype="dashed", linewidth=.5)+
  labs(title = "Distribution of Chi-Squared Values",
       x = expression(chi^2),
       y = "Frequency") +
  theme_minimal()



```

