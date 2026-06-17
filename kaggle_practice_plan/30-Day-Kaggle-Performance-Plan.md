# 🏆 The 30-Day Kaggle Performance Plan

### *10 Minutes a Day to Level Up Your Data Science & Machine Learning Game*

> **Consultant's note:** I've coached a lot of people who "want to get good at Kaggle" but never actually move the needle. The reason is almost never talent — it's consistency and the *wrong kind of practice*. This plan is built around one promise: **just 10 focused minutes a day for 30 days.** Small, daily reps beat occasional weekend marathons every single time. Treat it like going to the gym — show up, do the rep, log it, repeat.

---

## 📋 Table of Contents

- [How to Use This Plan](#-how-to-use-this-plan)
- [Before You Start (Day 0 Setup)](#-before-you-start-day-0-setup)
- [The Daily Ritual](#-the-daily-ritual)
- [Week 1 — Foundations & Workflow](#-week-1--foundations--workflow)
- [Week 2 — Exploratory Data Analysis (EDA) & Feature Engineering](#-week-2--exploratory-data-analysis-eda--feature-engineering)
- [Week 3 — Modeling, Validation & Tuning](#-week-3--modeling-validation--tuning)
- [Week 4 — Competing, Ensembling & Sharing](#-week-4--competing-ensembling--sharing)
- [Progress Tracker](#-progress-tracker)
- [Golden Rules from a Kaggle Expert](#-golden-rules-from-a-kaggle-expert)
- [Resources](#-resources)

---

## 🎯 How to Use This Plan

1. **Pick a fixed time slot.** Same 10 minutes every day (morning coffee, lunch, before bed). Consistency > duration.
2. **One task per day.** Each day has a single, bite-sized objective. Don't skip ahead — the order is intentional.
3. **Set a timer for 10 minutes.** When it rings, stop. The goal is to build a *habit*, not to burn out. If you're in flow and want to continue, that's a bonus, not a requirement.
4. **Log it.** Tick the box in the [Progress Tracker](#-progress-tracker). The streak is your real opponent.
5. **Use one "anchor" competition.** Pick a beginner-friendly competition on Day 1 and keep returning to it all month so your learning compounds on a real dataset.

> 💡 **Recommended anchor competitions for beginners:**
> - [Titanic — Machine Learning from Disaster](https://www.kaggle.com/c/titanic) (classification)
> - [House Prices — Advanced Regression Techniques](https://www.kaggle.com/c/house-prices-advanced-regression-techniques) (regression)
> - [Spaceship Titanic](https://www.kaggle.com/c/spaceship-titanic) (classification, slightly richer)

---

## 🛠️ Before You Start (Day 0 Setup)

Do this once before Day 1 (give yourself ~30 min here, it's the only long session):

- [ ] Create a free [Kaggle account](https://www.kaggle.com/) and complete your profile.
- [ ] Verify your phone number to unlock **Notebooks (Kernels)** with free GPU/TPU.
- [ ] Browse the [Kaggle Learn](https://www.kaggle.com/learn) micro-courses — bookmark *Python*, *Pandas*, *Intro to ML*, and *Intermediate ML*.
- [ ] Join the anchor competition you picked above and download / attach its dataset.
- [ ] Star this document and the Progress Tracker. **You're ready.**

---

## ⏱️ The Daily Ritual

Every single day, follow this 3-step loop:

| Step | Time | Action |
| ---- | ---- | ------ |
| **1. Warm-up** | 1 min | Open your anchor notebook. Re-read yesterday's last cell. |
| **2. Today's rep** | 8 min | Do the day's single task below. |
| **3. Log & note** | 1 min | Tick the tracker. Write *one sentence* on what you learned. |

> The "one sentence" note is non-negotiable — it turns passive practice into active learning and becomes your personal playbook by Day 30.

---

## 📅 Week 1 — Foundations & Workflow

> **Theme:** Get comfortable in the Kaggle environment and the data science loop. No pressure to score well yet — we're building muscle memory.

| Day | 10-Minute Task | Outcome |
| --- | -------------- | ------- |
| **1** | Create a new Notebook in your anchor competition. Load the data with `pandas` (`pd.read_csv`) and run `df.head()`, `df.shape`. | You can open data on Kaggle. |
| **2** | Run `df.info()` and `df.describe()`. Identify data types and which columns have missing values. | You understand your dataset's skeleton. |
| **3** | Complete 1–2 lessons of the [Kaggle Learn: Pandas](https://www.kaggle.com/learn/pandas) course. | Sharper data wrangling. |
| **4** | Count missing values per column: `df.isnull().sum()`. Note the worst 3 offenders. | You know your data gaps. |
| **5** | Plot one histogram of the target variable (`df['target'].hist()`). What's the distribution? | First visual intuition. |
| **6** | Fork a popular public notebook from the competition's *Code* tab. Read it top to bottom. | Learn from the community. |
| **7** | **Review day.** Re-read your 6 one-sentence notes. Write a 3-bullet summary of the dataset. | Consolidation. |

---

## 📅 Week 2 — Exploratory Data Analysis (EDA) & Feature Engineering

> **Theme:** *"Better features beat better models."* This is where competitions are actually won. Spend your reps here.

| Day | 10-Minute Task | Outcome |
| --- | -------------- | ------- |
| **8** | Make a correlation heatmap (`sns.heatmap(df.corr())`). Spot the features most correlated with the target. | Find signal. |
| **9** | Handle missing values for ONE column (mean/median/mode fill, or a "missing" flag). | First cleaning win. |
| **10** | Encode ONE categorical column (`pd.get_dummies` or `LabelEncoder`). | Models can read categories. |
| **11** | Create ONE new feature from existing columns (e.g., ratio, sum, group, date part). | Feature engineering rep. |
| **12** | Plot the new feature vs. the target. Did it add signal? Keep or discard. | Validate your idea. |
| **13** | Complete 1–2 lessons of [Kaggle Learn: Feature Engineering](https://www.kaggle.com/learn/feature-engineering). | New techniques. |
| **14** | **Review day.** List every cleaning + feature step you've done as a reusable checklist. | Your preprocessing pipeline v1. |

---

## 📅 Week 3 — Modeling, Validation & Tuning

> **Theme:** Trust your validation, not the leaderboard. A reliable cross-validation (CV) score is your superpower.

| Day | 10-Minute Task | Outcome |
| --- | -------------- | ------- |
| **15** | Train a baseline model (`LogisticRegression` or `LinearRegression`). Get any score. | You have a baseline! |
| **16** | Train a tree model (`RandomForest` or `XGBoost`/`LightGBM`). Compare to baseline. | Stronger model. |
| **17** | Set up proper **K-Fold cross-validation** (`cross_val_score`, k=5). | Trustworthy scoring. |
| **18** | Read about the competition's **evaluation metric**. Make sure you're optimizing the right thing. | Stop optimizing blindly. |
| **19** | Look at **feature importance** from your tree model. Drop the 2 weakest features and re-score. | Lean, mean model. |
| **20** | Tune ONE hyperparameter (e.g., `max_depth` or `n_estimators`). Note the effect. | Tuning intuition. |
| **21** | **Review day.** Record your best CV score and the exact config that produced it. | Reproducible best run. |

---

## 📅 Week 4 — Competing, Ensembling & Sharing

> **Theme:** Ship it, learn in public, and squeeze out the last few points. This is what separates spectators from competitors.

| Day | 10-Minute Task | Outcome |
| --- | -------------- | ------- |
| **22** | Generate predictions on the test set and create `submission.csv`. | First artifact. |
| **23** | **Make your first official submission.** Check the public leaderboard. | You're officially on the board! 🎉 |
| **24** | Compare your leaderboard score to your CV score. Big gap = overfitting or leakage — investigate. | CV vs. LB discipline. |
| **25** | Read the top public notebook's *discussion*. Steal ONE idea and try it. | Learn from the best. |
| **26** | Train a 2nd different model and **average** the two predictions (simple ensemble). Submit. | Ensembling basics. |
| **27** | Clean up your notebook: add markdown, headings, comments. Make it readable. | Portfolio-quality work. |
| **28** | **Publish your notebook publicly** and write a short description. Share your approach. | Build your reputation. |
| **29** | Write down your top 3 lessons of the month and your next 3 ideas to try. | Forward momentum. |
| **30** | **Celebrate & plan.** Pick your next competition and re-start this loop with harder goals. | Habit locked in. 🏅 |

---

## ✅ Progress Tracker

Tick a box each day. **Don't break the chain.**

```
Week 1:  [ ] 1   [ ] 2   [ ] 3   [ ] 4   [ ] 5   [ ] 6   [ ] 7
Week 2:  [ ] 8   [ ] 9   [ ] 10  [ ] 11  [ ] 12  [ ] 13  [ ] 14
Week 3:  [ ] 15  [ ] 16  [ ] 17  [ ] 18  [ ] 19  [ ] 20  [ ] 21
Week 4:  [ ] 22  [ ] 23  [ ] 24  [ ] 25  [ ] 26  [ ] 27  [ ] 28
Finish:  [ ] 29  [ ] 30
```

| Metric | Day 1 | Day 15 | Day 30 |
| ------ | ----- | ------ | ------ |
| Best CV score | | | |
| Best leaderboard score | | | |
| Notebooks published | | | |
| Current streak 🔥 | | | |

---

## 💎 Golden Rules from a Kaggle Expert

These are the principles I repeat to every person I consult. Internalize them:

1. **Consistency beats intensity.** 10 minutes daily for a month will take you further than one 8-hour binge.
2. **Trust your cross-validation, not the public leaderboard.** The public LB is a small, noisy slice. A robust CV is your compass.
3. **Spend most of your time on the data, not the model.** Feature engineering and clean validation win competitions; fancy models are the last 10%.
4. **Always have a baseline.** You can't measure improvement without a starting point.
5. **Read the discussion forums.** The best ideas in any competition are shared openly — your job is to read, adapt, and apply them.
6. **Submit early and often.** A mediocre submission today beats a perfect one that never ships.
7. **Learn in public.** Publishing notebooks forces clarity, builds your reputation, and attracts collaborators.
8. **One change at a time.** Change one thing, measure, keep or discard. This is how you build real intuition.
9. **Beware data leakage.** If your score looks too good, you're probably leaking future/target info. Be suspicious of perfection.
10. **Progress, not perfection.** A "Novice" who shows up every day becomes an "Expert" faster than a genius who shows up twice.

---

## 📚 Resources

| Resource | What it's for |
| -------- | ------------- |
| [Kaggle Learn](https://www.kaggle.com/learn) | Free, hands-on micro-courses (Python, Pandas, ML, FE, more). |
| [Kaggle Competitions](https://www.kaggle.com/competitions) | Browse & join challenges (filter by "Getting Started"). |
| [Kaggle Datasets](https://www.kaggle.com/datasets) | Free datasets for practice and side projects. |
| [Kaggle Notebooks](https://www.kaggle.com/code) | Read top-voted code from grandmasters. |
| [Kaggle Discussions](https://www.kaggle.com/discussions) | Where winning strategies are shared. |
| [scikit-learn docs](https://scikit-learn.org/stable/) | The ML toolkit you'll use most. |
| [XGBoost](https://xgboost.readthedocs.io/) / [LightGBM](https://lightgbm.readthedocs.io/) | Go-to gradient boosting libraries for tabular comps. |

---

> **Final word from your consultant:** Don't aim to be the best on Day 1. Aim to be *consistent*. Show up for these 10 minutes, every day, for 30 days — and by Day 30 you won't recognize the data scientist you've become. Now go set that timer. 🚀
