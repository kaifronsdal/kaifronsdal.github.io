---
layout: post
title: Can AI Self-Correct? Analyzing the Limitations of Large Language Models
---

_This post is a summary of [this](/writing/CS224N__Project_Final_Report.pdf) research paper writen as a final project for Stanford's CS 224N._

As models become more and more powerful, they have started to approach or exceed human-level performance on a variety of tasks. In order to train these models, researchers have been using increasingly large datasets and models. However, as these model approach human limits, it becomes harder and harder to improve them from human feedback.

The natural solution is to replace the human feedback with model feedback; that is, to have the model evaluate its own outputs and make necessary amendments. There are several varients to this process, including RLAIF and Constitutional AI, but the basic idea follows the following abstract process:

![RLAIF](../assets/can_llms_self_correct/Screenshot%202024-03-19%20at%203.21.02%20PM.png)

However, this method operates under the assumption that the feedback is actually aligned. Researchers have shown that these models can be easily swayed by biased feedback, which might hint at a fundamental deficit in their ability to self-correct and result in runaway feedback loops if used in an RLAIF pipeline. In this post we will explore the validity of the feedback step in this pipeline, and whether it is possible for AI to self-correct.

<!-- The concept of self-correction, in particular, offers the potential for enhancing the fundamental reasoning abilities of Large Language Models (LLMs). -->

<!-- For years, artificial intelligence (AI) researchers have been refining a type of machine learning model known as large language models (LLMs). Their goal is to create AI that understands language at a human-like level, and in some respects, they have been astonishingly successful. These models can generate convincing text that appears almost human-crafted.  -->

<!-- However, while these models display intelligent behaviors, they also manifest unexpected and incorrect behaviors. They might produce responses that seem sensible at first glance but are fundamentally incorrect or nonsensical upon closer inspection. To address this issue, researchers have been exploring avenues to improve LLMs’ reasoning abilities. One promising method is self-correction, where a model evaluates its own outputs and makes necessary amendments.

In this post, we delve into the viability of self-correction in the domain of mathematics. In particular, we examine whether these AI models can detect errors in mathematical reasoning and calculation, and whether this ability can be improved with fine-tuning. -->

## The Challenges of Self-Correction

The idea behind self-correction is relatively simple. If you give an LLM a mathematical problem, it should be capable of identifying any errors in its own solution, just like a human might. In an ideal world, it could review the problem, correct the error, and generate a more accurate solution.

Unfortunately, it’s not quite that simple.

Recent results suggest that while LLMs are fairly good at correcting several reasoning tasks, they struggle to identify mistakes in mathematical reasoning. Intrestingly, they are often capable of correcting their errors when the errors are explicitly pointed out, but struggle to identify these errors themselves. Thus the question arises: why do these models struggle to find their own mistakes?

## Base Models are Overconfident

We begin our investigation with LLaMa-2 7B. Due to the sensitivity of model performance to specific prompts, we tried a wide range of prompts across three categories:
 - **Confidence**: We asked the model to evaluate it's confidence the given solution was correct.
 - **Mistake**: We asked the model to find mistakes in the solutions.
 - **Check**: We asked the model to double check all of the calculations.

Additionally, we tried several different rating scales
 - **Percent**: the models rates its confidence in the correctness of the solution on a scale from 0 to 100\%
 - **Rating**: the model rates the accuracy of the solution on a scale from 1-5
 - **Correctness**: the model outputs ``correct'' or ``incorrect.''
 - **Confidence Level**: the model outputs its confidence as one of "very confident," "confident", "somewhat confident", "not very confident", or "not confident"

We found that there was not a massive difference in performance between the four different scales. It is important to note that we rescaled each of the rating scales to be between 0 and 100 so it was easier to compare between the prompts. 




<!-- Recent results suggest that LLMs struggle to identify mistakes independently. They are often capable of correcting their errors when the errors are explicitly pointed out, but struggle to identify these errors themselves. Moreover, these models can be easily swayed by feedback, which might hint at a fundamental deficit in their ability to self-correct. -->

<!-- The popular method of human feedback—where human quality assessments of AI outputs act as a reward signal to optimize model performance—has been effective in mitigating some of these issues. However, given the massive compute requirements for training powerful models and the cost of collecting human feedback, it’s not a sustainable solution. -->

<!-- To address this, researchers have been developing augmentation techniques to reduce the reasoning errors of AI models. One of the key techniques explored is self-correction. -->

## Uncovering the Limitations

In our investigation, we found that smaller LLMs exhibit limitations in accurately assessing mathematical arguments. Fine-tuning for self-correction improved their self-calibration—in other words, curbing their overconfidence. However, it fell short in enhancing their ability to differentiate correct solutions from incorrect ones.

Through a detailed analysis of correct and incorrect solutions, we discovered that these models struggle to identify granular calculation mistakes such as carry errors and missing signs. They are, however, more adept at correcting high-level reasoning mistakes.

Performed on smaller AIs due to computational constraints, these findings indicate that self-correction appears to be a complex behavior that these models fundamentally lack.

## Where to next?

Our findings suggest the need for future research to delve deeper into more specific error types produced by these models to see if some can be mitigated.

We also recommend future experiments involving larger models to determine whether our hypothesis—that self-correction is an emergent behavior—holds true. Considering the complexities of language and mathematics, it may be that self-correction is a higher-order cognitive process, requiring a more sophisticated level of understanding than currently possible with these models.

In sum, the process of making AI systems capable of correcting their own mistakes is much more complex than we might have first imagined. The results of our research highlight the importance of further evaluation and improvement in the quest for self-correcting AI.

To learn more about our research in the finer details, you can read the full paper [here]().