# Model selection

## The question
The model predicts which customers are likely to cancel (churn), so the company can contact them with a retention offer. A missed churner means a lost customer and lost revenue; a false alarm only means an unnecessary discount, so missing churners is the worse mistake.

## Candidates
| Candidate | Why I tried it | AUC | F1 | F1 per slice (lowest) | p95 latency (ms) | Size (KB) | MLflow run ID |
|---|---|---|---|---|---|---|---|
| logreg_balanced | Simple, fast baseline; `class_weight="balanced"` so it stops ignoring the minority churn class | 0.797 | 0.589 | 0.333 (two-year) | 2.8 | 4 | bb312df2d2a74903bbfd507237680775 |
| random_forest_balanced | Many trees catch non-linear patterns; 150 trees, depth 8, min 5 customers per leaf to avoid memorizing; balanced | 0.768 | 0.549 | 0.400 (two-year) | 10.0 | 1522 | 8c50e3e6d5434a679c2d3b47dce704f2 |
| gradient_boosting | Usually the strongest family; 200 trees at learning rate 0.05; no class_weight option, so it tests whether a stronger model beats handling the imbalance | 0.759 | 0.421 | 0.364 (two-year) | 3.1 | 268 | d34494fe2b7049e4984470e9371343ba |

## The champion
I picked logreg_balanced. It has the best F1 (0.589) and AUC (0.797), and it is also the fastest (2.8 ms) and smallest (4 KB). Balancing the classes mattered more than model complexity: gradient boosting, without balancing, had the lowest F1 (0.421).
What I accepted losing: on the weakest slice (two-year contracts) it is the worst of the three, 0.333 versus 0.400 for the random forest. I accepted this because that slice has very few churners in the test set, so the gap is about one customer and is mostly noise, while the overall F1 gap is measured on all customers. It is also the best on the other two slices, including month-to-month (0.626 versus 0.571), the group where most churners are. Balanced weights also raise more false alarms (lower precision).

## My promotion rules
- Latency budget (p95): 25 ms, because the API answers one customer per request and the prediction must stay a small part of the response time; the champion runs in 2.8 ms on my laptop, and CI runners are slower, so 25 ms leaves a large safety margin.
- Minimum gain to replace the champion (F1 points): 0.01, because small F1 differences can come from noise in the test split; a new model must be clearly better, not just luckier.

## What I would check next
Customers on two-year contracts rarely churn, so that slice has few churners in the test set and its F1 is unstable. I would also test the champion on newer data (data/churn_v2.csv) to see whether performance holds when customer behaviour changes.
