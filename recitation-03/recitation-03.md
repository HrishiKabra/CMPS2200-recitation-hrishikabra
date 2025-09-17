# CMPS 2200  Recitation 03

**Name (Team Member 1):** Hrishi Kabra  
**Name (Team Member 2):** Petra Radmanovic



## Analyzing a recursive, parallel algorithm


You were recently hired by Netflix to work on their movie recommendation
algorithm. A key part of the algorithm works by comparing two users'
movie ratings to determine how similar the users are. For example, to
find users similar to Mary, we start by having Mary rank all her movies.
Then, for another user Joe, we look at Joe's rankings and count how
often his pairwise rankings disagree with Mary's:

|      | Beetlejuice | Batman | Jackie Brown | Mr. Mom | Multiplicity |
| ---- | ----------- | ------ | ------------ | ------- | ------------ |
| Mary | 1           | 2      | 3            | 4       | 5            |
| Joe  | 1           | **3**  | **4**        | **2**   | 5            |

Here, Joe (somehow) liked *Mr. Mom* more than *Batman* and *Jackie
Brown*, so the number of disagreements is 2:
(3 <->  2, 4 <-> 2). More formally, a
disagreement occurs for indices (i,j) when (j > i) and
(value[j] < value[i]).

When you arrived at Netflix, you were shocked (shocked!) to see that
they were using this O(n^2) algorithm to solve the problem:



``` python
def num_disagreements_slow(ranks):
    """
    Params:
      ranks...list of ints for a user's move rankings (e.g., Joe in the example above)
    Returns:
      number of pairwise disagreements
    """
    count = 0
    for i, vi in enumerate(ranks):
        for j, vj in enumerate(ranks[i:]):
            if vj < vi:
                count += 1
    return count
```

``` python 
>>> num_disagreements_slow([1,3,4,2,5])
2
```

Armed with your CMPS 2200 knowledge, you quickly threw together this
recursive algorithm that you claim is both more efficient and easier to
run on the giant parallel processing cluster Netflix has.

``` python
def num_disagreements_fast(ranks):
    # base cases
    if len(ranks) <= 1:
        return (0, ranks)
    elif len(ranks) == 2:
        if ranks[1] < ranks[0]:
            return (1, [ranks[1], ranks[0]])  # found a disagreement
        else:
            return (0, ranks)
    # recursion
    else:
        left_disagreements, left_ranks = num_disagreements_fast(ranks[:len(ranks)//2])
        right_disagreements, right_ranks = num_disagreements_fast(ranks[len(ranks)//2:])
        
        combined_disagreements, combined_ranks = combine(left_ranks, right_ranks)

        return (left_disagreements + right_disagreements + combined_disagreements,
                combined_ranks)

def combine(left_ranks, right_ranks):
    i = j = 0
    result = []
    n_disagreements = 0
    while i < len(left_ranks) and j < len(right_ranks):
        if right_ranks[j] < left_ranks[i]: 
            n_disagreements += len(left_ranks[i:])   # found some disagreements
            result.append(right_ranks[j])
            j += 1
        else:
            result.append(left_ranks[i])
            i += 1
    
    result.extend(left_ranks[i:])
    result.extend(right_ranks[j:])
    print('combine: input=(%s, %s) returns=(%s, %s)' % 
          (left_ranks, right_ranks, n_disagreements, result))
    return n_disagreements, result

```

```python
>>> num_disagreements_fast([1,3,4,2,5])
combine: input=([4], [2, 5]) returns=(1, [2, 4, 5])
combine: input=([1, 3], [2, 4, 5]) returns=(1, [1, 2, 3, 4, 5])
(2, [1, 2, 3, 4, 5])
```

As so often happens, your boss demands theoretical proof that this will
be faster than their existing algorithm. To do so, complete the
following:

a) Describe, in your own words, what the `combine` method is doing and
what it returns.

- The `combine` method takes two sorted lists (`left_ranks` and `right_ranks`) and merges them into one sorted list
- During the merge process, it identifies and counts disagreements between elements from the two lists
- A disagreement occurs when an element from the right list is smaller than an element from the left list
- When such a disagreement is found, it counts ALL remaining elements in the left list as disagreements with that right element
- This approach works because both input lists are already sorted, so if right[j] $<$ left[i], then right[j] is also less than all elements left[i+1], left[i+2], etc.
- The method returns both the total count of disagreements and the merged sorted list

b) Write the work recurrence formula for `num_disagreements_fast`. Please explain how do you have this.

- $W(n) = 2W(\frac{n}{2}) + O(n)$
- At each recursive level, we divide the problem into two subproblems of size $\frac{n}{2}$, giving us $2W(\frac{n}{2})$
- The combine operation performs a linear scan through both lists, resulting in $O(n)$ work per call
- The recursive calls operate on each half independently, and the work outside recursion (splitting and merging) is $O(n)$


c) Solve this recurrence using any method you like. Please explain how do you have this.

- The solution is $W(n) = O(n log n)$
- At the top level, we perform $n$ work
- At the next level, we have two subproblems of size $\frac{n}{2}$, so the work is $2 × \frac{n}{2} = n$
- This pattern continues: at each level, the total work remains $n$
- The recursion tree has $log_2 n$ levels total
- Therefore, the total work is $n × log_2 n = O(n log n)$


d) Assuming that your recursive calls to `num_disagreements_fast` are
done in parallel, write the span recurrence for your algorithm. Please explain how do you have this.

- $S(n) = S(\frac{n}{2}) + O(n)$
- The span considers only the longest path through the computation when executed in parallel
- Since the recursive calls run simultaneously, we only need to account for one branch: $S(\frac{n}{2})$
- The key difference from work is that we don't multiply by 2, as both recursive calls happen concurrently

e) Solve this recurrence using any method you like. Please explain how do you have this.

- Solving this recurrence: $n + \frac{n}{2} + \frac{n}{4} + ... = O(n)$
- This forms a geometric series that converges to $O(n)$
- Asymptotically, the dominant term is $O(n)$, so $S(n) = O(n)$
- The constant factors don't affect the asymptotic complexity

f) If `ranks` is a list of size n, Netflix says it will give you
lg(n) processors to run your algorithm in parallel. What is the
upper bound on the runtime of this parallel implementation? (Hint: assume a Greedy
Scheduler). Please explain how do you have this.

- Using Brent's Theorem with a greedy scheduler: $T_p ≤ \frac{W}{p} + S$ where $p = log(n)$ processors
- We have $W = O(n log n)$ and $S = O(n)$
- Substituting: $T_p ≤ \frac{n log n}{log n} + n = n + n = 2n$
- Asymptotically, $2n = O(n)$, so the upper bound runtime is $O(n)$


