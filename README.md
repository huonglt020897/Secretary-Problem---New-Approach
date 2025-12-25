*This repository was part of the assignment for Data-driven Leadership course in Business Intelligence master program.*

## **1. Problem Introduction**

The classical Secretary Problem illustrates how to balance exploration (observing early candidates) and exploitation (deciding when to stop and select). Under the traditional objective — **selecting the best candidate** — the optimal strategy is to explore about the first one third of candidates and then exploit on the first thereafter arriving outdoing the best candidate in the explored pool. This means that in expectation the selected and best candidate will arrive two thirds into the pool.

However, this idealized setup does not fully match realistic decision-making. In practice, selecting the absolute best is rarely necessary; candidates ranked second, third, or fourth may be practically interchangeable. In large pools, top candidates tend to be very similar in quality, making the strict best-choice objective unnecessarily rigid.

This assignment shifts the objective from selecting the single best candidate to **selecting a “good enough” candidate**, such as someone in the **top 1%, 5%, or 10%**.

The simulation study investigates:

- Whether the classical **exploration strategy** should be revised under a good-enough objective.

- How the acceptance threshold affects **search time**.

- How far the **selected candidate falls short of the true best**.

- How **pool size** (100, 500, 1000) influences these patterns.

Overall, the goal is to understand **how optimal stopping behavior changes** when the decision criterion becomes more flexible and realistic.

---
## **2. Step-by-Step Solution and Results**

### **2.1 Define selection method**

**Best-Explored Rule** (same as classic problem)

- Observe the initial exploration phase (fraction of the pool).
- Select the first candidate who exceeds the best quality observed during exploration.
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/53b55115-035e-4545-9bb7-0a8dcde7fa85" />


**Top-k Threshold Rule**

- Exploration data are used to estimate the empirical k-percentile threshold (e.g., top 1%, 5%, 10%).
- Select the first candidate whose quality exceeds the estimated top-k threshold.
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/692382b1-ff62-49f3-8031-2e3dd718efee" />

**Rank-Based Rule**

- Observe the initial exploration pool.
- When each new candidate appears, their quality is compared against all candidates observed so far, and a real-time rank is computed.
- Select the first candidate whose rank places within the top-k percent of the observed set.
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/40bae1ab-101a-4665-99db-e07bf611de86" />

---

### **2.2 Define performance metrics to measure performance**

To evaluate strategy effectiveness, several performance metrics were computed for each simulation run:

- **Success Rate**: Whether the selected candidate belongs to the true top-k% of the entire pool.

- **Gap to Best Candidate**: Difference between the quality of the best candidate and the selected candidate.

- **Gap to k-Threshold**: Difference between the top-k threshold and the selected candidate.

- **Selected Rank**: True ranking position of the chosen candidate.

- **Search Time**: Fraction of the pool processed before a selection is made.


---

### **2.3 Build a core simulation function**

- Create a function with 4 parameters
```
def simulate_one_new_problem(
    num_candidates=100, 
    strategy_fraction=1/np.e, 
    k_percent=0.01, 
    selection_method='first_above_k_threshold'):
    simulation_result = []
```

- Generate a sequence of candidate qualities from a uniform(0,1) distribution.
```
    quality_scores=np.random.uniform(0, 1, num_candidates)
```

- Compute relevant exploration statistics:

```
    # Identify the best candidate's quality score
    true_best = np.max(quality_scores)

    # Determine the k-th threshold quality score
    sorted_scores = sorted(quality_scores, reverse=True)
    k = int(num_candidates * k_percent)
    k_threshold = sorted_scores[k-1]

    # Determine the number of candidates to explore based on the strategy fraction and the best explored quality score
    explore_size = int(num_candidates * strategy_fraction)
    explored_scores = quality_scores[:explore_size]
    selected_candidate = quality_scores[-1]  # Default to last candidate if none selected earlier
    total_search = explore_size # Count of candidates evaluated after exploration
```

- Apply the chosen selection rule in the exploitation phase.

Top-k Threshold Rule

```
    if selection_method == 'first_above_k_threshold':
        estimated_k_threshold = np.quantile(explored_scores, 1 - k_percent)
        # Select the first candidate above the estimated k-th threshold
        for i in quality_scores[explore_size:]:
            total_search += 1
            if i >= estimated_k_threshold:
                selected_candidate = i
                break
```

Rank-Based Rule
```
    elif selection_method == 'rank-based':
        best_explored = np.max(explored_scores)
        seen_scores = explored_scores.copy()
        for i in quality_scores[explore_size:]:
            total_search += 1
            seen_scores = np.append(seen_scores, i)
            # rank i-th candidate among seen candidates
            sorted_seen = sorted(seen_scores, reverse=True)
            rank_in_seen = sorted_seen.index(i) + 1
            # compute top-k among seen pool
            k_seen = max(int(len(seen_scores) * k_percent), 1)
            # Select the first candidate who is in top-k of seen candidates or above the best explored
            if rank_in_seen <= k_seen:  # realistic rank rule
                selected_candidate = i
                break
```

Best-Explored Rule

```
    else:  # 'first_above_best_explored' - original strategy
        best_explored = np.max(explored_scores)
        # Select the candidate with the highest quality score after the exploration phase
        for i in quality_scores[explore_size:]:
            total_search += 1
            if i >= best_explored:
                selected_candidate = i
                break

```
- Record outcome variables (success, rank, gaps, search time, fallback).
```
    # Check rank of the selected candidate
    rank = (quality_scores > selected_candidate).sum()+1

    #Append the results to the simulation_results list
    simulation_result.append({
        'pool_size': num_candidates,
        'strategy_fraction': strategy_fraction,
        'top_k_threshold': k_threshold,
        'selected_candidate': selected_candidate,
        'selected_rank': rank,
        'is_top_k_candidate': selected_candidate >= k_threshold,
        'is_best_candidate': selected_candidate == true_best,
        'gap_to_best': (true_best - selected_candidate) / true_best,
        'gap_to_k_threshold': (k_threshold - selected_candidate) / k_threshold,
        'num_search': total_search,
        'search_time': total_search / num_candidates,
    })
    return pd.DataFrame(simulation_result)

```



---
### **2.4 Run the simulation across different parameters**

1000 simulations were computed for each configuration, details as below:
- Total number of candidates: ***num_candidates = [100, 500, 1000]***
- The percentage of exploration pool: ***strategy_fraction = [0.1, 0.2, 0.3, 1/np.e, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9]***
- Acceptance criteria: ***k_percent = [0.01, 0.05, 0.1]***
- Selection method: 
  - ***'rank-based method'***
  - ***'first_above_best_explored'***
  - ***'first_above_k_threshold'***


---

## **3. Results & Interpretation**
<img width="2126" height="1180" alt="image" src="https://github.com/user-attachments/assets/3a0fb5cf-ce14-4a90-900a-6171e22732da" />

<img width="2126" height="1180" alt="image" src="https://github.com/user-attachments/assets/116f2f7d-d5ba-48c4-b657-ed840ee05661" />

### **3.1. Is the current selection strategy good enough? Is there a need for revising the selection strategy? How does the strategy affect the search time and gap between selected candidate to the best one?**

***The classical strategy is not good enough and should be revised.***

Across all pool sizes and acceptance thresholds, the classical strategy (**first_above_best_explored**) performs systematically worse than the two revised strategies:

- It delivers lower success rates, especially when search time approaches 1.0.

- It produces the largest gap to the best candidate, often selecting weak late-arriving candidates.

- Its performance deteriorates sharply with longer search time, while the revised strategies remain stable.

Both revised strategies—especially the rank-based rule—substantially improve success rates and keep the gap-to-best near zero, indicating a clear need to revise the selection rule.

**Overall interpretation**

The classical strategy was optimal for selecting the single best candidate, but with the new “good-enough” objective, it becomes too conservative and frequently ends up selecting late-arriving low-quality candidates.
Thus, revising the strategy is necessary. Both revised strategies, especially the rank-based approach, significantly improve performance.

### **3.2. How the acceptance threshold affects search time?**
Across columns (1%, 5%, 10%), several patterns emerge:

**(A) Higher k makes the problem easier**

When k increases from 1% → 5% → 10%, all strategies show:

- Higher success rates

- Lower gap to the best candidate

- Flatter curves (less sensitivity to search time)

**(B) Rank-based strategy becomes nearly perfect**

For k = 10%, rank-based success rates approach 1.0 across almost the entire search-time range in medium and large pools.

**Interpretation**
Relaxing the decision criterion (moving from "top 1%" to "top 10%") reduces risk and substantially boosts performance, but the relative ordering of the strategies remains the same:
Rank-based > k-threshold > classical.

### **3.3. How pool size (100, 500, 1000) influences these patterns?**

- **Small pools (100)**: noisy, unstable, strategies look similar.

- **Medium pools (500)**: clearer separation; rank-based begins to dominate.

- **Large pools (1000)**: very strong patterns; rank-based achieves almost perfect success, while the classical strategy collapses near the end of the search.

**Interpretation**

Larger pools make exploration more informative, strategy performance more consistent and amplify the advantage of the revised strategies.
The rank-based strategy scales best with pool size because it uses real-time updates, allowing it to adapt as information accumulates.

---

## **4. Discussion and Conclusion**

- When the goal changes from selecting the absolute best to selecting a “good enough” candidate, the **classical strategy becomes inefficient**.
- The k-threshold and rank-based strategies align better with the revised objective, especially **rank-based delivers consistently high success rates and minimal gap**, thanks to its real-time updating.
- **Higher acceptance thresholds (k%) make the problem easier**, with all strategies improving, and the rank-based method approaching near-perfect performance.
- Larger pool sizes reduce randomness and **magnify differences** between strategies: adaptive methods improve, while the classical strategy deteriorates.
