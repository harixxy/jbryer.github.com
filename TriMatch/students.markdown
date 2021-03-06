---
layout: mathjax	
title: TriMatch
subtitle: Propensity Score Matching for Non-Binary Treatments
published: true
status: publish
submenu: trimatch
---
 
#### Example One: New Student Outreach
 


 
The data in example one (`data(students)`) represent newly enrolled students in a distance education program. By the nature of the program all students are considered part-time. Moreover, enrollment in the institution does not necessarily mean students are making active progress towards degree completion. To address this issue the institution began an outreach program whereby academic advisors would regularly contact new students within the first six months of enrollment until either six months have passed, or the student enrolled in some credit earning activity (generally a course or an examination for credit). Since randomization to receive the outreach was not possible, a comparison group was defined as students who enrolled six months prior to the start of the outreach. The treatment group is identified as students who enrolled six months after the start of the outreach and who received at least one academic advisor contact. 
 
Covariates for estimating propensity scores were retrieved from the student information system. The dependent variable of interest is the number of credits attempted within the first seven months of enrollment.
 
During the implementation phase it was identified that the outreach being conducted was substantially different between the two academic advisors responsible for the treatment. As such, it became necessary to treat each academic advisor as a separate treatment. The analysis of propensity score models for more than two groups (i.e. two treatments and one control) has relied on conducting three separate analyses. We outline here an approach to conducting propensity score analysis with three groups. 
 

    data(students)
    names(students)

 
We will create a `treat` variable that identifies our three groups.
 

    treat <- students$TreatBy
    table(treat, useNA = "ifany")

    treat
       Control Treatment1 Treatment2 
           200         83         91 

    describeBy(students$CreditsAttempted, group = list(students$TreatBy), mat = TRUE, skew = FALSE)

       item     group1 var   n  mean    sd median trimmed   mad min max range     se
    11    1    Control   1 200 7.225 7.903      5   6.069 7.413   0  26    26 0.5588
    12    2 Treatment1   1  83 4.361 6.156      0   3.254 0.000   0  27    27 0.6757
    13    3 Treatment2   1  91 5.923 7.148      3   4.767 4.448   0  25    25 0.7493

 
The following boxplot shows unadjusted results.
 

    ggplot(students, aes(x = TreatBy, y = CreditsAttempted, colour = TreatBy)) + geom_boxplot() + 
        coord_flip() + geom_jitter()

![plot of chunk boxplot](/images/trimatch/boxplot.png) 

 
#### Estimate Propensity Scores
 
The `trips` function will estimate three propensity score models.
 

    form <- ~Military + Income + Employment + NativeEnglish + EdLevelMother + EdLevelFather + HasAssocAtEnrollment + 
        Ethnicity + Gender + Age
    tpsa <- trips(students, students$TreatBy, formu = form)

    Warning: glm.fit: fitted probabilities numerically 0 or 1 occurred

    Warning: glm.fit: fitted probabilities numerically 0 or 1 occurred

 
 

    head(tpsa)

      id      treat model1 model2 model3       ps1       ps2 ps3 strata1 strata2 strata3
    1  1    Control  FALSE  FALSE     NA 1.462e-01 1.687e-01  NA       1       1    <NA>
    2  2 Treatment1   TRUE     NA   TRUE 8.304e-01        NA   1       5    <NA>       5
    3  3    Control  FALSE  FALSE     NA 3.934e-01 3.693e-01  NA       4       4    <NA>
    4  4    Control  FALSE  FALSE     NA 2.965e-08 1.779e-08  NA       1       1    <NA>
    5  5    Control  FALSE  FALSE     NA 2.965e-08 1.779e-08  NA       1       1    <NA>
    6  6    Control  FALSE  FALSE     NA 2.220e-16 1.650e-08  NA       1       1    <NA>

    (p <- plot(tpsa))

![plot of chunk psaestimates](/images/trimatch/psaestimates.png) 

 
#### Matched Triplets
 

    tmatch <- trimatch(tpsa, exact = students[, c("Military", "DegreeLevel")])

 
Triangle plot of the results. We can see how the propensity scores translate from one model to another.
 

    head(tmatch)

      Treatment2 Treatment1 Control      D.m3     D.m1    D.m2  Dtotal
    1        321        105      91 0.0004357 0.010718 0.01161 0.02276
    2        342        326     198 0.0150613 0.002718 0.04715 0.06493
    3         30        310     203 0.0408941 0.004873 0.02937 0.07514
    4         95        310     248 0.0454191 0.020088 0.01894 0.08444
    5        104        105     129 0.0007764 0.070095 0.01655 0.08742
    6        352        105     109 0.0091104 0.013030 0.06693 0.08907

    plot(tmatch, rows = c(2), line.alpha = 1, draw.segments = TRUE)

![plot of chunk matches](/images/trimatch/matches.png) 

 
We can plot the distances. We can specify other calipers to see how may matched triplets we eliminate if we specify a small caliper to the `trimatch` function.
 

    plot.distances(tmatch, caliper = c(0.15, 0.2, 0.25))

    Standard deviations of propensity scores: 0.17 + 0.15 + 0.23 = 0.55
    Percentage of matches exceding a distance of 0.15 (caliper = 0.15): 11%
    Percentage of matches exceding a distance of 0.2 (caliper = 0.2): 0.56%
    Percentage of matches exceding a distance of 0.25 (caliper = 0.25): 0%

![plot of chunk distances](/images/trimatch/distances.png) 

 
The numbers on the left edge are the row numbers from `tmatch`. We can then use the `plot.triangle.matches` function with specifying the `rows` parameters to any or all of these values to investigate that matched triplet. The following figures shows that the large distances in due to the fact that only one data point has a very large propensity score in both model 1 and 2.
 

    tmatch[tmatch$Dtotal > 0.6, ]

        Treatment2 Treatment1 Control   D.m3   D.m1   D.m2 Dtotal
    179        295         50     184 0.1842 0.2494 0.1959 0.6294

    tmatch[466, ]

       Treatment2 Treatment1 Control D.m3 D.m1 D.m2 Dtotal
    NA       <NA>       <NA>    <NA>   NA   NA   NA     NA

    plot(tmatch, rows = c(466), line.alpha = 1, draw.segments = TRUE)

    Error: 'data' must be of a vector type

 
#### Checking balance.
 
 

    plot.balance(tmatch, students$Age, label = "Age")

    Using propensity scores from model 3 for evaluating balance.

    
    	Friedman rank sum test
    
    data:  Covariate and Treatment and ID 
    Friedman chi-squared = 10.39, df = 2, p-value = 0.00555
    
     Repeated measures ANOVA
    
         Effect DFn DFd    F        p p<.05     ges
    2 Treatment   2 356 7.01 0.001033     * 0.02293

![plot of chunk balance](/images/trimatch/balance1.png) 

    plot.balance(tmatch, students$Age, label = "Age", nstrata = 8)

    Using propensity scores from model 3 for evaluating balance.

    
    	Friedman rank sum test
    
    data:  Covariate and Treatment and ID 
    Friedman chi-squared = 10.39, df = 2, p-value = 0.00555
    
     Repeated measures ANOVA
    
         Effect DFn DFd    F        p p<.05     ges
    2 Treatment   2 356 7.01 0.001033     * 0.02293

![plot of chunk balance](/images/trimatch/balance2.png) 

    plot.balance(tmatch, students$Military, label = "Military", nstrata = 10)

    Using propensity scores from model 3 for evaluating balance.

    
    	Friedman rank sum test
    
    data:  Covariate and Treatment and ID 
    Friedman chi-squared = NaN, df = 2, p-value = NA

![plot of chunk balance](/images/trimatch/balance3.png) 

    plot.balance(tmatch, students$Gender, label = "Gender")

    Using propensity scores from model 3 for evaluating balance.

    
    	Friedman rank sum test
    
    data:  Covariate and Treatment and ID 
    Friedman chi-squared = 13.57, df = 2, p-value = 0.00113

![plot of chunk balance](/images/trimatch/balance4.png) 

    plot.balance(tmatch, students$Ethnicity, label = "Ethnicity")

    Using propensity scores from model 3 for evaluating balance.

    
    	Friedman rank sum test
    
    data:  Covariate and Treatment and ID 
    Friedman chi-squared = 0.215, df = 2, p-value = 0.8981

![plot of chunk balance](/images/trimatch/balance5.png) 

 

    covs <- students[, c("Military", "NativeEnglish", "Gender", "Age", "Income", "Employment", 
        "EdLevelMother", "EdLevelFather")]
    covs$Gender <- as.integer(covs$Gender)
    covs$Income <- as.integer(covs$Income)
    covs$Employment <- as.integer(covs$Employment)
    covs$EdLevelMother <- as.integer(covs$EdLevelMother)
    covs$EdLevelFather <- as.integer(covs$EdLevelFather)
    plot.multibalance(tpsa, covs, grid = TRUE)

![plot of chunk effectsizes](/images/trimatch/effectsizes.png) 

 
#### Examine unmatched students.
 

    # Look at the subjects that could not be matched
    unmatched <- attr(tmatch, "unmatched")
    nrow(unmatched)/nrow(tpsa) * 100

    [1] 53.74

    # Percentage of each group not matched
    table(unmatched$treat)/table(tpsa$treat) * 100

    
       Control Treatment1 Treatment2 
         66.50      42.17      36.26 

    unmatched[unmatched$treat != "Control", ]

         id      treat model1 model2 model3    ps1    ps2       ps3 strata1 strata2 strata3
    2     2 Treatment1   TRUE     NA   TRUE 0.8304     NA 1.000e+00       5    <NA>       5
    23   23 Treatment1   TRUE     NA   TRUE 1.0000     NA 1.000e+00       5    <NA>       5
    38   38 Treatment1   TRUE     NA   TRUE 0.5186     NA 5.457e-01       5    <NA>       3
    65   65 Treatment1   TRUE     NA   TRUE 0.7518     NA 1.000e+00       5    <NA>       5
    70   70 Treatment1   TRUE     NA   TRUE 0.3445     NA 6.562e-01       4    <NA>       4
    80   80 Treatment1   TRUE     NA   TRUE 0.5788     NA 8.082e-01       5    <NA>       5
    83   83 Treatment1   TRUE     NA   TRUE 0.7625     NA 1.000e+00       5    <NA>       5
    87   87 Treatment2     NA   TRUE  FALSE     NA 0.4371 2.259e-01    <NA>       5       1
    119 119 Treatment2     NA   TRUE  FALSE     NA 0.5751 1.641e-01    <NA>       5       1
    123 123 Treatment2     NA   TRUE  FALSE     NA 0.6137 1.316e-01    <NA>       5       1
    125 125 Treatment2     NA   TRUE  FALSE     NA 0.5315 1.413e-08    <NA>       5       1
    138 138 Treatment1   TRUE     NA   TRUE 0.4051     NA 6.757e-01       4    <NA>       5
    139 139 Treatment2     NA   TRUE  FALSE     NA 0.3737 3.892e-08    <NA>       4       1
    143 143 Treatment2     NA   TRUE  FALSE     NA 0.3551 6.714e-01    <NA>       4       4
    144 144 Treatment1   TRUE     NA   TRUE 0.5993     NA 7.979e-01       5    <NA>       5
    149 149 Treatment2     NA   TRUE  FALSE     NA 0.5527 2.139e-01    <NA>       5       1
    213 213 Treatment2     NA   TRUE  FALSE     NA 0.1445 5.528e-01    <NA>       1       4
    214 214 Treatment1   TRUE     NA   TRUE 0.5998     NA 8.225e-01       5    <NA>       5
    216 216 Treatment2     NA   TRUE  FALSE     NA 0.2992 3.410e-01    <NA>       3       2
    240 240 Treatment2     NA   TRUE  FALSE     NA 0.3969 2.137e-01    <NA>       4       1
    267 267 Treatment2     NA   TRUE  FALSE     NA 0.6377 2.153e-01    <NA>       5       1
    277 277 Treatment2     NA   TRUE  FALSE     NA 0.4930 1.852e-01    <NA>       5       1
    282 282 Treatment2     NA   TRUE  FALSE     NA 0.4925 3.043e-01    <NA>       5       2
    284 284 Treatment2     NA   TRUE  FALSE     NA 0.4910 2.230e-08    <NA>       5       1
    285 285 Treatment2     NA   TRUE  FALSE     NA 0.6094 1.195e-01    <NA>       5       1
    286 286 Treatment2     NA   TRUE  FALSE     NA 0.3559 2.083e-01    <NA>       4       1
    289 289 Treatment1   TRUE     NA   TRUE 0.2875     NA 3.651e-01       3    <NA>       2
    290 290 Treatment1   TRUE     NA   TRUE 0.2381     NA 5.260e-01       2    <NA>       3
    296 296 Treatment1   TRUE     NA   TRUE 0.1898     NA 1.883e-01       2    <NA>       1
    299 299 Treatment1   TRUE     NA   TRUE 0.3980     NA 4.123e-01       4    <NA>       2
    302 302 Treatment2     NA   TRUE  FALSE     NA 0.3743 6.308e-08    <NA>       4       1
    308 308 Treatment1   TRUE     NA   TRUE 0.3783     NA 7.631e-01       4    <NA>       5
    309 309 Treatment2     NA   TRUE  FALSE     NA 0.4612 3.000e-01    <NA>       5       2
    311 311 Treatment1   TRUE     NA   TRUE 0.4735     NA 5.857e-01       5    <NA>       4
    312 312 Treatment1   TRUE     NA   TRUE 0.5615     NA 6.758e-01       5    <NA>       5
    313 313 Treatment1   TRUE     NA   TRUE 0.2626     NA 5.589e-01       3    <NA>       4
    314 314 Treatment1   TRUE     NA   TRUE 0.5754     NA 7.471e-01       5    <NA>       5
    315 315 Treatment1   TRUE     NA   TRUE 0.1773     NA 3.214e-01       2    <NA>       2
    317 317 Treatment1   TRUE     NA   TRUE 0.4762     NA 8.001e-01       5    <NA>       5
    318 318 Treatment2     NA   TRUE  FALSE     NA 0.4372 5.909e-01    <NA>       5       4
    319 319 Treatment2     NA   TRUE  FALSE     NA 0.5894 1.260e-01    <NA>       5       1
    323 323 Treatment2     NA   TRUE  FALSE     NA 0.3476 2.220e-16    <NA>       4       1
    324 324 Treatment2     NA   TRUE  FALSE     NA 0.3517 6.188e-08    <NA>       4       1
    325 325 Treatment1   TRUE     NA   TRUE 0.5885     NA 7.360e-01       5    <NA>       5
    327 327 Treatment1   TRUE     NA   TRUE 0.5673     NA 5.496e-01       5    <NA>       3
    332 332 Treatment2     NA   TRUE  FALSE     NA 0.6212 3.449e-01    <NA>       5       2
    335 335 Treatment1   TRUE     NA   TRUE 1.0000     NA 1.000e+00       5    <NA>       5
    336 336 Treatment1   TRUE     NA   TRUE 0.6168     NA 5.726e-01       5    <NA>       4
    338 338 Treatment2     NA   TRUE  FALSE     NA 0.2720 4.562e-01    <NA>       2       3
    339 339 Treatment2     NA   TRUE  FALSE     NA 0.6239 1.097e-01    <NA>       5       1
    345 345 Treatment1   TRUE     NA   TRUE 0.3443     NA 7.315e-01       4    <NA>       5
    350 350 Treatment1   TRUE     NA   TRUE 0.3303     NA 7.407e-01       4    <NA>       5
    353 353 Treatment1   TRUE     NA   TRUE 0.5003     NA 8.108e-01       5    <NA>       5
    356 356 Treatment1   TRUE     NA   TRUE 0.3477     NA 5.409e-01       4    <NA>       3
    360 360 Treatment1   TRUE     NA   TRUE 0.2902     NA 3.978e-01       3    <NA>       2
    361 361 Treatment1   TRUE     NA   TRUE 0.3101     NA 6.108e-01       3    <NA>       4
    362 362 Treatment1   TRUE     NA   TRUE 0.4252     NA 5.813e-01       5    <NA>       4
    363 363 Treatment2     NA   TRUE  FALSE     NA 0.5417 3.601e-01    <NA>       5       2
    364 364 Treatment2     NA   TRUE  FALSE     NA 0.2766 5.596e-01    <NA>       2       4
    365 365 Treatment1   TRUE     NA   TRUE 0.8931     NA 1.000e+00       5    <NA>       5
    366 366 Treatment2     NA   TRUE  FALSE     NA 0.2139 6.202e-01    <NA>       2       4
    367 367 Treatment1   TRUE     NA   TRUE 0.3604     NA 1.000e+00       4    <NA>       5
    368 368 Treatment2     NA   TRUE  FALSE     NA 0.4665 2.543e-01    <NA>       5       2
    369 369 Treatment1   TRUE     NA   TRUE 0.4517     NA 1.000e+00       5    <NA>       5
    370 370 Treatment2     NA   TRUE  FALSE     NA 0.3318 6.000e-01    <NA>       3       4
    372 372 Treatment2     NA   TRUE  FALSE     NA 0.5336 1.678e-01    <NA>       5       1
    373 373 Treatment2     NA   TRUE  FALSE     NA 0.5506 1.345e-01    <NA>       5       1
    374 374 Treatment2     NA   TRUE  FALSE     NA 0.6451 2.147e-01    <NA>       5       1

 
We can create a triangle plot of only the unmatched students by subsetting `tpsa` with those students in the `unmatched` data frame.
 

    plot(tpsa[tpsa$id %in% unmatched$id, ])

![plot of chunk plotunmatched](/images/trimatch/plotunmatched.png) 

 
#### Loess Plot
 

    plot.loess3(tmatch, students$CreditsAttempted, plot.points = geom_jitter, ylab = "Credits Attempted")

    geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use
    'method = x' to change the smoothing method.

![plot of chunk loess](/images/trimatch/loess1.png) 

    plot.loess3(tmatch, students$CreditsAttempted, plot.points = geom_jitter, ylab = "Credits Attempted", 
        points.alpha = 0.5, plot.connections = TRUE)

    geom_smooth: method="auto" and size of largest group is <1000, so using loess. Use
    'method = x' to change the smoothing method.

![plot of chunk loess](/images/trimatch/loess2.png) 

 
#### Parrellel Plot
 

    plot.parallel(tmatch, students$CreditsAttempted)

![plot of chunk merge](/images/trimatch/merge.png) 

 
#### Friedman Rank Sum Test
 

    tmatch.out <- merge(tmatch, students$CreditsAttempted)
    outcomes <- grep(".out$", names(tmatch.out), perl = TRUE)
    tmatch.out$id <- 1:nrow(tmatch.out)
    out <- melt(tmatch.out[, c(outcomes, which(names(tmatch.out) == "id"))], id.vars = "id")
    names(out) <- c("ID", "Treatment", "Outcome")
    head(out)

      ID      Treatment Outcome
    1  1 Treatment2.out      17
    2  2 Treatment2.out       0
    3  3 Treatment2.out       0
    4  4 Treatment2.out      13
    5  5 Treatment2.out       0
    6  6 Treatment2.out       6

    set.seed(2112)
    friedman.test(Outcome ~ Treatment | ID, out)

    
    	Friedman rank sum test
    
    data:  Outcome and Treatment and ID 
    Friedman chi-squared = 2.257, df = 2, p-value = 0.3235

 
#### Repeated Measures ANOVA
 

    rmanova <- ezANOVA(data = out, dv = Outcome, wid = ID, within = Treatment)

    Warning: Converting "ID" to factor for ANOVA.

    print(rmanova)

    $ANOVA
         Effect DFn DFd     F      p p<.05      ges
    2 Treatment   2 356 1.993 0.1378       0.007381
    
    $`Mauchly's Test for Sphericity`
         Effect      W       p p<.05
    2 Treatment 0.9674 0.05342      
    
    $`Sphericity Corrections`
         Effect    GGe  p[GG] p[GG]<.05    HFe  p[HF] p[HF]<.05
    2 Treatment 0.9685 0.1394           0.9789 0.1389          

 
#### Pairwise Wilcoxon Rank Sum Tests
 

    pairwise.wilcox.test(x = out$Outcome, g = out$Treatment, paired = TRUE, p.adjust.method = "bonferroni")

    
    	Pairwise comparisons using Wilcoxon signed rank test 
    
    data:  out$Outcome and out$Treatment 
    
                   Treatment2.out Treatment1.out
    Treatment1.out 0.094          -             
    Control.out    1.000          0.533         
    
    P value adjustment method: bonferroni 

 
#### Posthoc *t*-tests
 

    (t1 <- t.test(x = tmatch.out$Treatment1.out, y = tmatch.out$Control.out, paired = TRUE))

    
    	Paired t-test
    
    data:  tmatch.out$Treatment1.out and tmatch.out$Control.out 
    t = -1.536, df = 178, p-value = 0.1262
    alternative hypothesis: true difference in means is not equal to 0 
    95 percent confidence interval:
     -2.7950  0.3481 
    sample estimates:
    mean of the differences 
                     -1.223 

    (t2 <- t.test(x = tmatch.out$Treatment2.out, y = tmatch.out$Control.out, paired = TRUE))

    
    	Paired t-test
    
    data:  tmatch.out$Treatment2.out and tmatch.out$Control.out 
    t = 0.2968, df = 178, p-value = 0.7669
    alternative hypothesis: true difference in means is not equal to 0 
    95 percent confidence interval:
     -1.420  1.923 
    sample estimates:
    mean of the differences 
                     0.2514 

    (t3 <- t.test(x = tmatch.out$Treatment2.out, y = tmatch.out$Treatment1.out, paired = TRUE))

    
    	Paired t-test
    
    data:  tmatch.out$Treatment2.out and tmatch.out$Treatment1.out 
    t = 2.04, df = 178, p-value = 0.04283
    alternative hypothesis: true difference in means is not equal to 0 
    95 percent confidence interval:
     0.04813 2.90159 
    sample estimates:
    mean of the differences 
                      1.475 

 
#### Boxplot of differences
 

    plot.boxdiff(tmatch, students$CreditsAttempted)

![plot of chunk boxplotdiff](/images/trimatch/boxplotdiff.png) 

