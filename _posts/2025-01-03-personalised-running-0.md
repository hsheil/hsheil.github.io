---
layout: post
title:  "Predict running PBs and generate personalised training plans - can ChatGPT do it?"
date:   2025-01-03 15:39:00 +0100
categories: running
excerpt: How good are LLMs like ChatGPT and DeepSeek for PB prediction and personalised training plans?
---



# Motivation

Just two questions to see if you will find this blog post interesting:

1. How fast can you run now?

2. How fast could the _best_ version of you run?

For any given distance, every runner has an accurate answer to the first question while most amateur / recreational runners have no idea on the second.

And for a lot of runners, it's not a burning question either - it's about running further, work / life balance or other very good reasons. But if you _are_ interested, then read on..

# Personalisation for Running

Runners improve by training, and the web is full of training plans (and strongly held opinions). These plans revolve around psychologically important milestones - a sub-40 10k, a sub-20 5k, sub-18 5k and so on. But because they are off the shelf, by definition they cannot take your exact stats into account. They are general and generic, and not personalised. This implies they are probably sub-optimal / inefficient. Lastly, how do you know if the training plan and goal is unachievable or too easy? Can we do better?

# What do we need to model running performance?

The main variables that predict running performance are:

1. Age.
2. Gender.
3. Some measure of aerobic and anaerobic fitness - provided either as a [VO2 max](https://en.wikipedia.org/wiki/VO2_max) or [VDOT](https://vdoto2.com/calculator/) score.
4. Ideally, some previous performances (to [ground](https://en.wikipedia.org/wiki/Symbol_grounding_problem) the prediction model).

I will add a fifth variable to this list because biomechanically it is important for running:

4. Weight.

So we would like a Machine Learning model that can use the information encoded in these variables to predict how much you could improve by and also give a training roadmap to get there.

As a quick experiment, let's not even write any code to build the first model candidate (Model 1). [Large Language Models](https://en.wikipedia.org/wiki/Large_language_model) (LLMs) should be at least as good as the unfiltered internet for a  task like this as long as we construct a good prompt to elicit a structured, stable response. We will use [ChatGPT](https://chatgpt.com) and one of the strongest open-source models [DeepSeek](https://chat.deepseek.com) to test. Both are trained on massive compute with trillions of tokens of text, and a percentage of these tokens will contain / encode the __collective running wisdom__ currently available on the internet.

# A simple prompt

Here's the template prompt to drive our LLM-based Model 1 - replace the values in brackets "{}" with your own data: 

```
Consider a {age_category} {gender} runner with a vo2 max of {vo2_max} and weighing {weight} kg.
5k PB time is {5k_time} and 10k PB time is {10k_time}.
Maximum distance run per week is {max_distance} km, usually {avg_distance} km.
Estimate the PBs achievable in both 5k and 10k assuming good training in the next 12 months.
Use age and peer group rankings to formulate your response and provide relevant training sessions in CSV format to paste into a spreadsheet.
```

The per-runner variables to be subbed in  are:
```
{age_category}
{gender}
{vo2_max}
{weight}
{5k_time}
{10k_time}
```

So for me the template becomes:
```
Consider a V45 male runner with a vo2 max of 59 and weighing 76 kg.
5k PB time is 17:45 and 10k PB time is 37:01.
Maximum distance run per week is 75 km, usually 50 km.
Estimate the PBs achievable in both 5k and 10k assuming good training in the next 12 months.
Use age and peer group rankings to formulate your response and provide relevant training sessions in CSV format to paste into a spreadsheet.
```

# Actual DeepSeek Output for the Simple Prompt

To estimate the potential 5k and 10k personal bests (PBs) for a 45-year-old male runner with a VO2 max of 59 and a current 5k PB of 17:45 and 10k PB of 37:01, we can use the following approach:

### 1. **VO2 Max and Performance Potential**
   - A VO2 max of 59 is quite high for a 45-year-old male, indicating strong aerobic capacity. This suggests there is room for improvement in race times with proper training.
   - Based on Jack Daniels' VDOT tables, a VO2 max of 59 corresponds to a 5k time of approximately **17:00** and a 10k time of approximately **35:30**. These are theoretical estimates and assume optimal training and race conditions.

### 2. **Age-Graded Performance**
   - Using age-grading calculators, a 45-year-old male with a 5k time of 17:45 has an age-graded performance of approximately **82%**, and a 10k time of 37:01 is around **81%**. These are strong performances but indicate room for improvement.
   - With consistent training, it is reasonable to aim for an age-graded performance of **85-90%**, which would correspond to a 5k time of **16:30-16:45** and a 10k time of **34:30-35:00**.

### 3. **Training Volume and Intensity**
   - The runner currently averages 50 km per week with a peak of 75 km. To improve, they should aim to gradually increase their weekly mileage to **70-90 km**, with a focus on structured workouts.
   - Key training sessions should include:
     - **Interval training** (e.g., 6-8 x 1 km at 5k race pace with 90 seconds recovery).
     - **Tempo runs** (e.g., 20-30 minutes at 10k race pace or slightly slower).
     - **Long runs** (e.g., 15-20 km at an easy pace, incorporating strides or surges).
     - **Recovery runs** (easy pace to promote recovery and build aerobic base).

### 4. **Projected PBs**
   - With consistent training and proper recovery, the runner could realistically achieve:
     - **5k PB: 16:30-16:45**
     - **10k PB: 34:30-35:00**

### 5. **Training Plan (CSV Format)**
Below is a sample 12-week training plan in CSV format to help the runner achieve these goals. This plan assumes a gradual increase in volume and intensity.

```csv
Week,Session 1,Session 2,Session 3,Session 4,Session 5,Session 6,Total Volume (km)
1,6x1k @5k pace (90s recovery),8 km easy run,5 km tempo run @10k pace,Rest,10 km long run,5 km recovery run,50
2,8x800m @5k pace (90s recovery),8 km easy run,6 km tempo run @10k pace,Rest,12 km long run,5 km recovery run,55
3,5x1k @5k pace (60s recovery),10 km easy run,7 km tempo run @10k pace,Rest,14 km long run,6 km recovery run,60
4,6x1k @5k pace (90s recovery),8 km easy run,5 km tempo run @10k pace,Rest,10 km long run,5 km recovery run,50
5,8x1k @5k pace (90s recovery),10 km easy run,8 km tempo run @10k pace,Rest,16 km long run,6 km recovery run,65
6,10x800m @5k pace (60s recovery),10 km easy run,8 km tempo run @10k pace,Rest,18 km long run,6 km recovery run,70
7,6x1k @5k pace (90s recovery),12 km easy run,10 km tempo run @10k pace,Rest,20 km long run,8 km recovery run,75
8,8x1k @5k pace (90s recovery),12 km easy run,10 km tempo run @10k pace,Rest,20 km long run,8 km recovery run,80
9,5x1k @5k pace (60s recovery),10 km easy run,8 km tempo run @10k pace,Rest,18 km long run,6 km recovery run,70
10,6x1k @5k pace (90s recovery),12 km easy run,10 km tempo run @10k pace,Rest,20 km long run,8 km recovery run,80
11,8x1k @5k pace (90s recovery),12 km easy run,10 km tempo run @10k pace,Rest,20 km long run,8 km recovery run,85
12,4x1k @5k pace (90s recovery),8 km easy run,5 km tempo run @10k pace,Rest,12 km long run,5 km recovery run,50
```

### 6. **Key Notes**
   - **Recovery**: Ensure adequate rest and recovery, especially after hard sessions.
   - **Nutrition**: Focus on proper fueling and hydration to support training.
   - **Consistency**: Stick to the plan and avoid overtraining to reduce injury risk.

By following this plan and staying consistent, the runner should be able to achieve the projected PBs within 12 months.

# Actual ChatGPT Output for the Simple Prompt

### Achievable Personal Bests (PBs)  

**Constraints:**  
- Maximum training volume: ~75 km/week (typically 50 km/week).  
- **VO2max**: 59.  
- **Age Group**: V45 (45–49 years old).  
- **Current PBs**:  
  - **5k**: 17:45.  
  - **10k**: 37:01.  

With a consistent 50–75 km/week and strategic training:  
- **5k PB**: **16:55–17:10**.  
- **10k PB**: **35:45–36:15**.  

These estimates reflect what is achievable within your training volume constraints, aligning with age-group peer performance and physiological capacity.

---

### Training Plan Design  

**Focus Areas:**  
1. **Mileage Efficiency:** Prioritize quality over quantity.  
2. **Threshold Development:** Tempo runs for lactate threshold improvement.  
3. **VO2max Gains:** Shorter, faster intervals.  
4. **Long Run Consistency:** Enhance endurance without overloading.  
5. **Injury Prevention:** Strength, mobility, and recovery strategies.  

---

### Training Plan (CSV Format)

```csv
Week,Day,Session,Intensity,Details
1,Monday,Easy Run,60-70% HRmax,8-10 km recovery pace.
1,Tuesday,Threshold Intervals,80-85% HRmax,"5x1 km @ 10k pace, 90s jog recovery."
1,Wednesday,Recovery Run,60-65% HRmax,6 km easy pace.
1,Thursday,VO2max Intervals,90-95% HRmax,"6x800m @ 3k pace, 2:00 recovery."
1,Friday,Rest or Active Recovery,Low,"Optional cycling, yoga, or light strength training."
1,Saturday,Long Run,65-75% HRmax,"14-18 km steady-state with last 3 km at threshold."
1,Sunday,Short Repeats,95-100% HRmax,"12x200m @ mile pace, 90s jog recovery."
2,Monday,Easy Run,60-70% HRmax,10 km recovery pace.
2,Tuesday,Hill Repeats,85-95% HRmax,"8x200m uphill, jog back recovery."
2,Wednesday,Recovery Run,60-65% HRmax,6-8 km relaxed pace.
2,Thursday,Tempo Run,80-85% HRmax,"8 km @ threshold pace, moderate effort."
2,Friday,Strength Training,N/A,Core work, lunges, and mobility exercises.
2,Saturday,Long Run,65-75% HRmax,16-20 km comfortable.
2,Sunday,VO2max Intervals,90-95% HRmax,"4x1200m @ 5k pace, 2:30 rest."
3,Monday,Easy Run,60-70% HRmax,8-10 km recovery.
3,Tuesday,Progression Run,70-85% HRmax,"10 km building from easy to 10k pace."
3,Thursday,Interval Session,90-95% HRmax,"5x1k @ 3k pace, 2:30 rest."
...
```

**Weekly Mileage Range:** ~50–70 km, scaling with recovery needs.  

Would you like to refine this further for specific goals or preferences?


# Conclusion

The Model 1 output is ok to adequate but the LLMs suffer from the following drawbacks:

1. Neither model is grounded enough to use the age factor tables published by the [World Masters Athletics](https://world-masters-athletics.org) association (WMA). And when I modified the prompt to explicitly request this, the model either got the calculation wrong or used the wrong value from the lookup table (all LLMs are known to perform poorly at explicit math tasks).
2. DeepSeek quotes the VDOT table from Jack Daniels but provides the wrong value to base its prediction on.
2. The predictions are ok, but the training plans are still __generic and not personalised__. 
3. By definition, there is no feedback loop between the training plan and the model.

Some interesting notes comparing DeepSeek and ChatGPT side-by-side:
1. DeepSeek seems to have a better opinion of my running performance than ChatGPT!
2. ChatGPT places more emphasis on the constraint of 50 - 75 km per week, possibly explaining why it is more conservative with predictions.


A better model (Model 2, Model 3) would need to bring these major upgrades over Model 1 in my opinion:

1. Use the [2023 WMA tables](https://world-masters-athletics.org/wp-content/uploads/2023/02/2023-WMA-Appendix-B.pdf) explicitly and directly (and possibly the parkrun calculation) to better ground the future predictions on PBs.
2. Generate more than one prediction with probabilities attached to each - i.e. achieve a 1% PB with 95% likelihood, 5% PB with 80% etc.
2. Build an internal view of where the runner is in his / her running career. A more complete history of PBs / big milestone runs could encode a lot of what is needed here.
2. Scale up to half-marathon and marathon distances and down to 3,000m, 1 mile etc.
2. Generate a training plan at the start of a training block and then be able to access runner progress as the block progresses to successively refine the probability of the initial prediction being achieved. Apps like [TrainAsONE](https://trainasone.com) and [AI Endurance](https://aiendurance.com) claim to do this, but reviews online are mixed..
