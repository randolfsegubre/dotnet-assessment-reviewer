# Mock Exam 3 — Coding Challenges (10 Problems)

> ⏱️ **Time limit: 90 minutes** | Similar to Coderbyte coding assessment format  
> Write working code. Test your solution mentally before checking.

---

## Challenge 1: Two Sum
**Difficulty:** Easy | **Language:** C#

Given an array of integers and a target sum, return the indices of the two numbers that add up to the target. Assume exactly one solution exists.

```
Input: nums = [2, 7, 11, 15], target = 9
Output: [0, 1]  (nums[0] + nums[1] = 2 + 7 = 9)
```

<details>
<summary>✅ Solution</summary>

```csharp
// O(n) solution using HashMap
public int[] TwoSum(int[] nums, int target)
{
    var seen = new Dictionary<int, int>(); // value → index

    for (int i = 0; i < nums.Length; i++)
    {
        int complement = target - nums[i];
        
        if (seen.ContainsKey(complement))
            return [seen[complement], i];
        
        seen[nums[i]] = i;
    }

    return []; // no solution (problem guarantees one exists)
}

// Tests:
// TwoSum([2,7,11,15], 9)   → [0,1]
// TwoSum([3,2,4], 6)       → [1,2]
// TwoSum([3,3], 6)          → [0,1]
```

**Time:** O(n) | **Space:** O(n)
</details>

---

## Challenge 2: Valid Brackets
**Difficulty:** Easy/Medium | **Language:** C#

Given a string containing only `(`, `)`, `{`, `}`, `[`, `]`, determine if the input string is valid. A string is valid if: open brackets are closed by the same type of brackets in the correct order.

```
Input: "({[]})"   → true
Input: "([)]"     → false
Input: "{[]}"     → true
Input: "{"        → false
```

<details>
<summary>✅ Solution</summary>

```csharp
public bool IsValid(string s)
{
    var stack = new Stack<char>();
    var pairs = new Dictionary<char, char>
    {
        { ')', '(' },
        { '}', '{' },
        { ']', '[' }
    };

    foreach (char c in s)
    {
        if (!pairs.ContainsKey(c)) // it's an opening bracket
        {
            stack.Push(c);
        }
        else // it's a closing bracket
        {
            if (stack.Count == 0 || stack.Pop() != pairs[c])
                return false;
        }
    }

    return stack.Count == 0; // all brackets were closed
}
```

**Time:** O(n) | **Space:** O(n)
</details>

---

## Challenge 3: Find Duplicate in Array
**Difficulty:** Medium | **Language:** C#

Given an array of integers where every integer appears twice except for one, find that one.

```
Input: [4, 1, 2, 1, 2]  → 4
Input: [2, 2, 1]         → 1
```

<details>
<summary>✅ Solution</summary>

```csharp
// XOR solution — O(n) time, O(1) space
// XOR properties: a ^ a = 0, a ^ 0 = a
public int FindSingle(int[] nums)
{
    int result = 0;
    foreach (int n in nums)
        result ^= n; // duplicates cancel out
    return result;
}

// Alternative: HashSet O(n) time, O(n) space
public int FindSingleHashSet(int[] nums)
{
    var seen = new HashSet<int>();
    foreach (int n in nums)
    {
        if (!seen.Add(n)) // Add returns false if already exists
            seen.Remove(n);
    }
    return seen.First();
}
```
</details>

---

## Challenge 4: Reverse Words in a String
**Difficulty:** Easy | **Language:** C#

Given a string, reverse the order of words. Words are separated by spaces. Trim leading/trailing spaces and collapse multiple spaces.

```
Input:  "  hello world  "      → "world hello"
Input:  "a good   example"     → "example good a"
```

<details>
<summary>✅ Solution</summary>

```csharp
public string ReverseWords(string s)
{
    var words = s.Split(' ', StringSplitOptions.RemoveEmptyEntries);
    Array.Reverse(words);
    return string.Join(' ', words);
}

// Or using LINQ:
public string ReverseWordsLinq(string s)
    => string.Join(' ', s.Split(' ', StringSplitOptions.RemoveEmptyEntries).Reverse());
```
</details>

---

## Challenge 5: Group Anagrams
**Difficulty:** Medium | **Language:** C#

Given an array of strings, group the anagrams together. Return the groups in any order.

```
Input: ["eat","tea","tan","ate","nat","bat"]
Output: [["bat"],["nat","tan"],["ate","eat","tea"]]
```

<details>
<summary>✅ Solution</summary>

```csharp
public IList<IList<string>> GroupAnagrams(string[] strs)
{
    var groups = new Dictionary<string, List<string>>();

    foreach (string word in strs)
    {
        // Sort characters → anagrams produce the same key
        char[] chars = word.ToCharArray();
        Array.Sort(chars);
        string key = new string(chars);

        if (!groups.ContainsKey(key))
            groups[key] = [];
        
        groups[key].Add(word);
    }

    return groups.Values.Cast<IList<string>>().ToList();
}

// Tests:
// ["eat","tea","tan","ate","nat","bat"] → [["eat","tea","ate"],["tan","nat"],["bat"]]
```

**Time:** O(n × k log k) where k = max word length | **Space:** O(n × k)
</details>

---

## Challenge 6: Calculate Net Worth Over Time
**Difficulty:** Medium | **Language:** C# — Domain-specific

Given a list of `Transaction` objects, calculate the running net worth (cumulative balance) per day, starting from an initial balance. Return a list of `(Date, Balance)` ordered by date.

```csharp
public record Transaction(DateTime Date, decimal Amount, bool IsIncome);
// Amount is always positive; IsIncome = true means credit, false = debit

// Input:
// Initial: 10000
// Transactions: (Jan 1, 5000, income), (Jan 3, 2000, expense), (Jan 3, 1000, income)
// 
// Output:
// (Jan 1, 15000), (Jan 3, 14000)
```

<details>
<summary>✅ Solution</summary>

```csharp
public IEnumerable<(DateTime Date, decimal Balance)> CalculateRunningBalance(
    IEnumerable<Transaction> transactions,
    decimal initialBalance)
{
    decimal balance = initialBalance;

    // Group by date and process in order
    var byDate = transactions
        .GroupBy(t => t.Date.Date) // group by calendar date
        .OrderBy(g => g.Key);

    foreach (var dateGroup in byDate)
    {
        foreach (var transaction in dateGroup)
        {
            balance += transaction.IsIncome ? transaction.Amount : -transaction.Amount;
        }

        yield return (dateGroup.Key, balance); // deferred — efficient for large datasets
    }
}

// LINQ version
public IList<(DateTime Date, decimal Balance)> CalculateRunningBalanceLinq(
    IEnumerable<Transaction> transactions,
    decimal initialBalance)
{
    return transactions
        .GroupBy(t => t.Date.Date)
        .OrderBy(g => g.Key)
        .Aggregate(
            seed: (balance: initialBalance, result: new List<(DateTime, decimal)>()),
            func: (acc, dateGroup) =>
            {
                decimal dayNet = dateGroup.Sum(t => t.IsIncome ? t.Amount : -t.Amount);
                decimal newBalance = acc.balance + dayNet;
                acc.result.Add((dateGroup.Key, newBalance));
                return (newBalance, acc.result);
            })
        .result;
}
```
</details>

---

## Challenge 7: Longest Substring Without Repeating Characters
**Difficulty:** Medium | **Language:** C#

Given a string, find the length of the longest substring without repeating characters.

```
Input: "abcabcbb" → 3  ("abc")
Input: "bbbbb"    → 1  ("b")
Input: "pwwkew"   → 3  ("wke")
```

<details>
<summary>✅ Solution</summary>

```csharp
// Sliding window approach — O(n)
public int LengthOfLongestSubstring(string s)
{
    var charIndex = new Dictionary<char, int>(); // char → last seen index
    int maxLen = 0;
    int left = 0; // left boundary of window

    for (int right = 0; right < s.Length; right++)
    {
        char c = s[right];
        
        // If char seen in current window, move left boundary past it
        if (charIndex.TryGetValue(c, out int lastIndex) && lastIndex >= left)
            left = lastIndex + 1;

        charIndex[c] = right;
        maxLen = Math.Max(maxLen, right - left + 1);
    }

    return maxLen;
}
```

**Time:** O(n) | **Space:** O(min(n, alphabet_size))
</details>

---

## Challenge 8: Implement a Simple LRU Cache
**Difficulty:** Hard | **Language:** C#

Implement a Least Recently Used (LRU) cache with `Get(key)` and `Put(key, value)` operations. Both must run in O(1). When capacity is exceeded, evict the least recently used item.

<details>
<summary>✅ Solution</summary>

```csharp
public class LRUCache
{
    private readonly int _capacity;
    private readonly Dictionary<int, LinkedListNode<(int Key, int Value)>> _cache;
    private readonly LinkedList<(int Key, int Value)> _order; // most recent at front

    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _cache = new Dictionary<int, LinkedListNode<(int, int)>>(capacity);
        _order = new LinkedList<(int, int)>();
    }

    public int Get(int key)
    {
        if (!_cache.TryGetValue(key, out var node))
            return -1; // not found

        // Move to front (most recently used)
        _order.Remove(node);
        _order.AddFirst(node.Value);
        _cache[key] = _order.First!;

        return node.Value.Value;
    }

    public void Put(int key, int value)
    {
        if (_cache.TryGetValue(key, out var existing))
        {
            _order.Remove(existing);
        }
        else if (_cache.Count >= _capacity)
        {
            // Evict least recently used (tail of linked list)
            var lru = _order.Last!;
            _order.RemoveLast();
            _cache.Remove(lru.Value.Key);
        }

        _order.AddFirst((key, value)); // add to front
        _cache[key] = _order.First!;
    }
}

// Usage:
// var cache = new LRUCache(2);
// cache.Put(1, 1); cache.Put(2, 2);
// cache.Get(1);    // returns 1, marks key 1 as recently used
// cache.Put(3, 3); // evicts key 2 (least recently used)
// cache.Get(2);    // returns -1 (evicted)
```

**Time:** O(1) for both Get and Put | **Space:** O(capacity)
</details>

---

## Challenge 9: Build a Simple Expression Evaluator
**Difficulty:** Medium | **Language:** C#

Evaluate a simple arithmetic expression string containing integers, `+`, `-`, `*`, `/`, and spaces. No parentheses.

```
Input: "3 + 2 * 2"    → 7
Input: " 3/2 "         → 1  (integer division)
Input: " 3+5 / 2 "    → 5
```

<details>
<summary>✅ Solution</summary>

```csharp
// Two-pass: handle * and / first (higher precedence)
public int Calculate(string s)
{
    var stack = new Stack<int>();
    int num = 0;
    char lastOp = '+'; // tracks the last operator seen

    for (int i = 0; i < s.Length; i++)
    {
        char c = s[i];

        if (char.IsDigit(c))
            num = num * 10 + (c - '0');

        if (!char.IsDigit(c) && c != ' ' || i == s.Length - 1)
        {
            // Process the number with the last operator
            switch (lastOp)
            {
                case '+': stack.Push(num); break;
                case '-': stack.Push(-num); break;
                case '*': stack.Push(stack.Pop() * num); break;
                case '/': stack.Push(stack.Pop() / num); break;
            }

            lastOp = c;
            num = 0;
        }
    }

    // Sum all values on the stack
    return stack.Sum();
}
```

**Time:** O(n) | **Space:** O(n)
</details>

---

## Challenge 10: Design a Rate Limiter
**Difficulty:** Hard | **Language:** C# — System Design

Implement a sliding window rate limiter that allows `maxRequests` per `windowSeconds`. Thread-safe.

```
RateLimiter limiter = new RateLimiter(maxRequests: 5, windowSeconds: 10);
limiter.IsAllowed("user123") → true (1st request)
// ... 4 more requests ...
limiter.IsAllowed("user123") → false (6th request in 10 sec window)
```

<details>
<summary>✅ Solution</summary>

```csharp
public class SlidingWindowRateLimiter
{
    private readonly int _maxRequests;
    private readonly TimeSpan _window;
    private readonly ConcurrentDictionary<string, Queue<DateTime>> _userRequests = new();
    private readonly object _lock = new();

    public SlidingWindowRateLimiter(int maxRequests, int windowSeconds)
    {
        _maxRequests = maxRequests;
        _window = TimeSpan.FromSeconds(windowSeconds);
    }

    public bool IsAllowed(string userId)
    {
        var now = DateTime.UtcNow;
        var windowStart = now - _window;

        var timestamps = _userRequests.GetOrAdd(userId, _ => new Queue<DateTime>());

        lock (timestamps) // lock per user, not global
        {
            // Remove expired timestamps
            while (timestamps.Count > 0 && timestamps.Peek() < windowStart)
                timestamps.Dequeue();

            if (timestamps.Count >= _maxRequests)
                return false; // rate limit exceeded

            timestamps.Enqueue(now);
            return true;
        }
    }
    
    // For cleanup — remove inactive users' state
    public void Cleanup()
    {
        var expiredKeys = _userRequests
            .Where(kvp => kvp.Value.Count == 0 || kvp.Value.Peek() < DateTime.UtcNow - _window)
            .Select(kvp => kvp.Key)
            .ToList();
        
        foreach (var key in expiredKeys)
            _userRequests.TryRemove(key, out _);
    }
}

// Middleware integration
public class RateLimitingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly SlidingWindowRateLimiter _limiter;

    public RateLimitingMiddleware(RequestDelegate next, SlidingWindowRateLimiter limiter)
    {
        _next = next; _limiter = limiter;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var userId = context.User.FindFirstValue("sub") ?? context.Connection.RemoteIpAddress?.ToString() ?? "anonymous";

        if (!_limiter.IsAllowed(userId))
        {
            context.Response.StatusCode = StatusCodes.Status429TooManyRequests;
            context.Response.Headers["Retry-After"] = "10";
            await context.Response.WriteAsJsonAsync(new { error = "Rate limit exceeded" });
            return;
        }

        await _next(context);
    }
}
```
</details>

---

## 📊 Score Guide

| Challenges Solved | Assessment |
|---|---|
| 9–10 | Excellent — Senior level |
| 7–8 | Good — Strong mid/senior |
| 5–6 | Average — Mid level |
| 3–4 | Below average — Junior/Mid |
| 0–2 | Needs more practice |

## Key Patterns Tested

| Pattern | Challenges |
|---|---|
| HashMap/Dictionary | 1, 5 |
| Stack | 2, 9 |
| Sliding Window | 7, 10 |
| XOR / Bit manipulation | 3 |
| Linked List | 8 |
| LINQ / Aggregation | 6 |
| String manipulation | 4, 7 |
| System design | 10 |
