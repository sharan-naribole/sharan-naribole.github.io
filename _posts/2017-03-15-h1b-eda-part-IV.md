---
title: 'H-1B Visa Petitions Data Analysis using R (Part IV): Kaggle Open Data'
date: 2017-03-15 12:00:00 -0600
permalink: /posts/2017/03/15/h1b-eda-part-III
tags:
  - data science
  - kaggle
redirect_from:
  - /2017/03/15/h1b-eda-part-III.html
toc: true
---

*This post is part of a series of blogs on exploration of H-1B visa petitions public dataset using R language.*

[Part I: Data Wrangling][h1b-part-I]

[Part II: Data Analysis][h1b-part-II]

[Part III: Shiny Web App][h1b-part-III]

[Part IV: Kaggle Open Data][h1b-part-IV]

## Part IV: Kaggle Open Data

In this blog, I describe briefly my experience publishing the [H-1B visa dataset on Kaggle][kaggle-h1b].

After I finished creating the Shiny web app and posted my blog on [NYC Data Science Academy][nyc-dsa], the blog was picked up by [R-bloggers][r-bloggers], the most popular blog aggregator for articles related to R language. I observed a positive interest from the readers to further explore this dataset. This enthusiasm was a key driver in publishing the H-1B visa dataset on Kaggle Datasets platform.

## Kaggle: Your Home for Data Science

According to Wikipedia,

> [Kaggle][kaggle-home] was founded as a platform for predictive modeling and analytics competitions on which companies and researchers post their data and statisticians and data miners from all over the world compete to produce the best models.

In 2017, Kaggle has not only become a central hub for Machine Learning competitions but also one of the best platforms for [open datasets][kaggle-datasets]! Personally, I love the [Kaggle kernels][kaggle-kernels] where you can explore codes and visualizations of fellow Kagglers and also share your own work. While getting my hands dirty with the famous [Titanic][kaggle-titanic] dataset, I picked up a ton of skills including XGBoost algorithm design, state-of-the-art stacking techniques and feature selection tricks just from reading the top Kaggle kernels and related forum discussion of the Titanic competition.

## Dataset of the Week!

![Kaggle H-1B Dataset of the Week](/images/h_1b_kaggle/h1_b_kaggle.png "Kaggle H-1B Dataset of the Week")

Two weeks after the [H-1B dataset][kaggle-h1b] was published, I was delighted to receive an email from Megan Risdal, Marketing Manager at Kaggle informing me that my dataset was chosen as Dataset of the Week for March 15 - March 16 2017. It will also be included in the first of Kaggle's new monthly [blog][kaggle-blog] series "Dataset of the Week” as well as the newsletter.

The dataset got promoted on Kaggle's social media including [Twitter][kaggle-twitter] and [Facebook][kaggle-facebook]. In the first two and half weeks of dataset getting published, there have nearly 1000 downloads and 56 kernels created to explore the dataset.

## Kaggle Dataset

The instructions provided while uploading to Kaggle were quite helpful. Due to the 500 MB limit on data upload, I made slight changes to the dataset I used for my own analysis and for the Shiny app. The columns in the dataset include:

1. **CASE_STATUS**: Status associated with the last significant event or decision. Valid values include “Certified,” “Certified-Withdrawn,” Denied,” and “Withdrawn”.

2. **EMPLOYER_NAME**: Name of employer submitting labor condition application.

3. **SOC_NAME**: Occupational name associated with the SOC_CODE. SOC_CODE is the occupational code associated with the job being requested for temporary labor condition, as classified by the Standard Occupational Classification (SOC) System.

4. **JOB_TITLE**: Title of the job

5. **FULL_TIME_POSITION**: Y = Full Time Position; N = Part Time Position

6. **PREVAILING_WAGE**: Prevailing Wage for the job being requested for temporary labor condition. The wage is listed at annual scale in USD. The prevailing wage for a job position is defined as the average wage paid to similarly employed workers in the requested occupation in the area of intended employment. The prevailing wage is based on the employer’s minimum requirements for the position.

7. **YEAR**: Year in which the H-1B visa petition was filed

8. **WORKSITE**: City and State information of the foreign worker's intended area of employment

9. **lon**: longitude of the Worksite

10. **lat**: latitude of the Worksite


## Conclusion and Future Work

This brings an end to this series on H-1B Visa Petitions Data Analysis using R. Hope you've enjoyed reading thus far and found something useful! Thanks

[reddit]: https://www.reddit.com/
[kaggle-facebook]:https://www.facebook.com/kaggle/photos/a.10150387148668464.377856.135534208463/10155150524548464/
[kaggle-twitter]: https://twitter.com/kaggle/status/842108218709037056
[kaggle-blog]: http://blog.kaggle.com/
[kaggle-h1b]: https://www.kaggle.com/nsharan/h-1b-visa
[kaggle-titanic]: https://www.kaggle.com/c/titanic
[kaggle-kernels]: https://www.kaggle.com/kernels
[kaggle-datasets]: https://www.kaggle.com/datasets/
[kaggle-home]: https://www.kaggle.com/
[r-bloggers]: https://www.r-bloggers.com/h-1b-visa-petitions-exploratory-data-analysis/
[h1b-part-I]: https://sharan-naribole.github.io/2017/02/24/h1b-eda-part-I.html
[h1b-part-II]: https://sharan-naribole.github.io/2017/02/26/h1b-eda-part-II.html
[h1b-part-III]:https://sharan-naribole.github.io/2017/02/28/h1b-eda-part-III.html
[h1b-part-IV]: https://sharan-naribole.github.io/2017/03/15/h1b-eda-part-IV.html
