# Coding Practice Interview Questions

## Question 1: Two Sum Problem

### The Question
> "Given an array of integers and a target sum, return the indices of two numbers that add up to the target. Can you solve this efficiently? What's the time and space complexity of your solution?"

### Key Points to Cover
- Brute force vs optimized approach
- Hash table for O(1) lookup
- Time/Space complexity trade-offs
- Edge cases (duplicates, no solution, negative numbers)

### Detailed Answer

**Brute Force Approach (O(n²)):**
Check every pair of numbers - simple but slow.

**Optimized Approach (O(n)):**
Use a dictionary to store numbers we've seen. For each number, check if its complement (target - current) exists in the dictionary.

### Code Example
```csharp
public class TwoSumSolution
{
    // Brute Force - O(n²) time, O(1) space
    public int[] TwoSumBruteForce(int[] nums, int target)
    {
        for (int i = 0; i < nums.Length; i++)
        {
            for (int j = i + 1; j < nums.Length; j++)
            {
                if (nums[i] + nums[j] == target)
                {
                    return [i, j];
                }
            }
        }
        return []; // No solution found
    }

    // Optimized - O(n) time, O(n) space
    public int[] TwoSum(int[] nums, int target)
    {
        var seen = new Dictionary<int, int>(); // value -> index

        for (int i = 0; i < nums.Length; i++)
        {
            int complement = target - nums[i];

            if (seen.TryGetValue(complement, out int complementIndex))
            {
                return [complementIndex, i];
            }

            // Handle duplicates: only store first occurrence
            seen.TryAdd(nums[i], i);
        }

        return []; // No solution found
    }
}

// Test cases
public class TwoSumTests
{
    [Theory]
    [InlineData(new[] { 2, 7, 11, 15 }, 9, new[] { 0, 1 })]
    [InlineData(new[] { 3, 2, 4 }, 6, new[] { 1, 2 })]
    [InlineData(new[] { 3, 3 }, 6, new[] { 0, 1 })]
    public void TwoSum_ReturnsCorrectIndices(int[] nums, int target, int[] expected)
    {
        var solution = new TwoSumSolution();
        var result = solution.TwoSum(nums, target);
        Assert.Equal(expected, result);
    }
}
```

### Communication Tactics
🎯 **Structure your answer**: "Let me start with the brute force approach, then optimize it. The brute force checks every pair in O(n²). We can do better with a hash table..."

💡 **Emphasize**: The trade-off between time and space - we're using extra memory to gain speed. This is a common pattern in algorithm optimization.

⚠️ **Avoid**: Jumping straight to the optimal solution without explaining your thought process. Interviewers want to see how you think.

---

## Question 2: Longest Substring Without Repeating Characters

### The Question
> "Given a string, find the length of the longest substring without repeating characters. For example, 'abcabcbb' should return 3 ('abc'). Walk me through your approach and analyze the complexity."

### Key Points to Cover
- Sliding window technique
- Hash set or dictionary for tracking characters
- Two-pointer approach
- How to handle edge cases (empty string, all same characters)

### Detailed Answer

**Approach: Sliding Window**
Maintain a window with two pointers (left and right). Expand the right pointer, and when we hit a duplicate, shrink from the left until the duplicate is removed.

**Why Sliding Window?**
- We need a contiguous sequence (substring)
- We want to maximize length
- We need to track uniqueness

### Code Example
```csharp
public class LongestSubstringSolution
{
    // Sliding Window with HashSet - O(n) time, O(min(m,n)) space
    // where m is the size of the character set
    public int LengthOfLongestSubstring(string s)
    {
        if (string.IsNullOrEmpty(s)) return 0;

        var charSet = new HashSet<char>();
        int maxLength = 0;
        int left = 0;

        for (int right = 0; right < s.Length; right++)
        {
            // Shrink window until duplicate is removed
            while (charSet.Contains(s[right]))
            {
                charSet.Remove(s[left]);
                left++;
            }

            charSet.Add(s[right]);
            maxLength = Math.Max(maxLength, right - left + 1);
        }

        return maxLength;
    }

    // Optimized with Dictionary (jump directly to position after duplicate)
    public int LengthOfLongestSubstringOptimized(string s)
    {
        if (string.IsNullOrEmpty(s)) return 0;

        var charIndex = new Dictionary<char, int>(); // char -> last seen index
        int maxLength = 0;
        int left = 0;

        for (int right = 0; right < s.Length; right++)
        {
            if (charIndex.TryGetValue(s[right], out int lastIndex))
            {
                // Jump left pointer past the duplicate
                left = Math.Max(left, lastIndex + 1);
            }

            charIndex[s[right]] = right;
            maxLength = Math.Max(maxLength, right - left + 1);
        }

        return maxLength;
    }
}

// Test cases
public class LongestSubstringTests
{
    [Theory]
    [InlineData("abcabcbb", 3)]  // "abc"
    [InlineData("bbbbb", 1)]     // "b"
    [InlineData("pwwkew", 3)]    // "wke"
    [InlineData("", 0)]          // empty
    [InlineData(" ", 1)]         // single space
    [InlineData("dvdf", 3)]      // "vdf" - tricky case
    public void LengthOfLongestSubstring_ReturnsCorrectLength(string s, int expected)
    {
        var solution = new LongestSubstringSolution();
        Assert.Equal(expected, solution.LengthOfLongestSubstring(s));
    }
}
```

### Communication Tactics
🎯 **Structure your answer**: "This is a classic sliding window problem. I'll maintain a window of unique characters and expand/contract it as I traverse the string..."

💡 **Emphasize**: Why sliding window works here - we're looking for contiguous elements (substring) and want to maximize while maintaining a constraint (uniqueness).

⚠️ **Avoid**: Using substring methods that create new strings - this wastes memory. Work with indices instead.

---

## Question 3: LINQ Grouping and Aggregation

### The Question
> "Given a list of orders with products and quantities, group them by category and calculate the total quantity and revenue per category. Show me how you'd write this in LINQ with both query and method syntax."

### Key Points to Cover
- GroupBy operation
- Aggregation functions (Sum, Count, Average)
- Anonymous types vs custom DTOs
- Query syntax vs method syntax
- Performance considerations

### Detailed Answer

**LINQ Approach:**
1. GroupBy the key field
2. Project each group into aggregated results
3. Consider ordering the results

**When to use which syntax:**
- Query syntax: Complex joins, multiple froms, let clauses
- Method syntax: Simple operations, chaining, better IntelliSense

### Code Example
```csharp
public class Order
{
    public int Id { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public string Category { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

public class CategorySummary
{
    public string Category { get; set; } = string.Empty;
    public int TotalQuantity { get; set; }
    public decimal TotalRevenue { get; set; }
    public int OrderCount { get; set; }
    public decimal AverageOrderValue { get; set; }
}

public class OrderAnalytics
{
    // Method Syntax - more common in production code
    public List<CategorySummary> GetCategorySummary_MethodSyntax(List<Order> orders)
    {
        return orders
            .GroupBy(o => o.Category)
            .Select(g => new CategorySummary
            {
                Category = g.Key,
                TotalQuantity = g.Sum(o => o.Quantity),
                TotalRevenue = g.Sum(o => o.Quantity * o.UnitPrice),
                OrderCount = g.Count(),
                AverageOrderValue = g.Average(o => o.Quantity * o.UnitPrice)
            })
            .OrderByDescending(c => c.TotalRevenue)
            .ToList();
    }

    // Query Syntax - sometimes more readable for complex queries
    public List<CategorySummary> GetCategorySummary_QuerySyntax(List<Order> orders)
    {
        var query = from order in orders
                    group order by order.Category into categoryGroup
                    let totalRevenue = categoryGroup.Sum(o => o.Quantity * o.UnitPrice)
                    orderby totalRevenue descending
                    select new CategorySummary
                    {
                        Category = categoryGroup.Key,
                        TotalQuantity = categoryGroup.Sum(o => o.Quantity),
                        TotalRevenue = totalRevenue,
                        OrderCount = categoryGroup.Count(),
                        AverageOrderValue = categoryGroup.Average(o => o.Quantity * o.UnitPrice)
                    };

        return query.ToList();
    }

    // Multiple aggregations with conditional logic
    public object GetAdvancedAnalytics(List<Order> orders)
    {
        return orders
            .GroupBy(o => o.Category)
            .Select(g => new
            {
                Category = g.Key,
                TotalRevenue = g.Sum(o => o.Quantity * o.UnitPrice),
                HighValueOrders = g.Count(o => o.Quantity * o.UnitPrice > 100),
                TopProduct = g.OrderByDescending(o => o.Quantity)
                              .First().ProductName,
                Products = g.Select(o => o.ProductName).Distinct().ToList()
            })
            .ToList();
    }
}

// Test
public class LinqTests
{
    [Fact]
    public void GetCategorySummary_GroupsCorrectly()
    {
        var orders = new List<Order>
        {
            new() { Id = 1, ProductName = "Laptop", Category = "Electronics", Quantity = 2, UnitPrice = 1000 },
            new() { Id = 2, ProductName = "Phone", Category = "Electronics", Quantity = 5, UnitPrice = 500 },
            new() { Id = 3, ProductName = "Shirt", Category = "Clothing", Quantity = 10, UnitPrice = 50 },
            new() { Id = 4, ProductName = "Pants", Category = "Clothing", Quantity = 3, UnitPrice = 75 },
        };

        var analytics = new OrderAnalytics();
        var result = analytics.GetCategorySummary_MethodSyntax(orders);

        Assert.Equal(2, result.Count);

        var electronics = result.First(c => c.Category == "Electronics");
        Assert.Equal(7, electronics.TotalQuantity);
        Assert.Equal(4500m, electronics.TotalRevenue); // 2*1000 + 5*500
    }
}
```

### Communication Tactics
🎯 **Structure your answer**: "I'll use GroupBy to partition the data, then aggregate each group. Let me show both syntaxes..."

💡 **Emphasize**: The difference between query and method syntax is purely stylistic - they compile to the same code. Choose based on readability.

⚠️ **Avoid**: Forgetting `ToList()` - LINQ is lazy, so without materialization you're just defining a query, not executing it. Also avoid multiple enumerations of IEnumerable.

---

## Question 4: Implement an LRU Cache

### The Question
> "Implement an LRU (Least Recently Used) cache with O(1) get and put operations. Explain your data structure choices and how they achieve the required time complexity."

### Key Points to Cover
- Dictionary for O(1) lookup
- Doubly linked list for O(1) insertion/removal
- Combined data structure
- Thread-safety considerations
- Capacity management

### Detailed Answer

**Key Insight:** We need two data structures working together:
1. **Dictionary**: O(1) key lookup
2. **Doubly Linked List**: O(1) removal from middle and insertion at head

When we access an item, we move it to the front. When we need space, we remove from the back.

### Code Example
```csharp
public class LRUCache<TKey, TValue> where TKey : notnull
{
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<CacheItem>> _cache;
    private readonly LinkedList<CacheItem> _lruList;

    private class CacheItem
    {
        public TKey Key { get; }
        public TValue Value { get; set; }

        public CacheItem(TKey key, TValue value)
        {
            Key = key;
            Value = value;
        }
    }

    public LRUCache(int capacity)
    {
        if (capacity <= 0)
            throw new ArgumentException("Capacity must be positive", nameof(capacity));

        _capacity = capacity;
        _cache = new Dictionary<TKey, LinkedListNode<CacheItem>>(capacity);
        _lruList = new LinkedList<CacheItem>();
    }

    public TValue? Get(TKey key)
    {
        if (!_cache.TryGetValue(key, out var node))
        {
            return default;
        }

        // Move to front (most recently used)
        MoveToFront(node);
        return node.Value.Value;
    }

    public void Put(TKey key, TValue value)
    {
        if (_cache.TryGetValue(key, out var existingNode))
        {
            // Update existing
            existingNode.Value.Value = value;
            MoveToFront(existingNode);
            return;
        }

        // Evict if at capacity
        if (_cache.Count >= _capacity)
        {
            EvictLeastRecentlyUsed();
        }

        // Add new item at front
        var newItem = new CacheItem(key, value);
        var newNode = _lruList.AddFirst(newItem);
        _cache[key] = newNode;
    }

    public bool TryGet(TKey key, out TValue? value)
    {
        if (_cache.TryGetValue(key, out var node))
        {
            MoveToFront(node);
            value = node.Value.Value;
            return true;
        }

        value = default;
        return false;
    }

    private void MoveToFront(LinkedListNode<CacheItem> node)
    {
        if (node != _lruList.First)
        {
            _lruList.Remove(node);
            _lruList.AddFirst(node);
        }
    }

    private void EvictLeastRecentlyUsed()
    {
        var lruNode = _lruList.Last;
        if (lruNode != null)
        {
            _cache.Remove(lruNode.Value.Key);
            _lruList.RemoveLast();
        }
    }

    public int Count => _cache.Count;
}

// Thread-safe version using ReaderWriterLockSlim
public class ThreadSafeLRUCache<TKey, TValue> where TKey : notnull
{
    private readonly LRUCache<TKey, TValue> _cache;
    private readonly ReaderWriterLockSlim _lock = new();

    public ThreadSafeLRUCache(int capacity)
    {
        _cache = new LRUCache<TKey, TValue>(capacity);
    }

    public TValue? Get(TKey key)
    {
        _lock.EnterUpgradeableReadLock();
        try
        {
            return _cache.Get(key);
        }
        finally
        {
            _lock.ExitUpgradeableReadLock();
        }
    }

    public void Put(TKey key, TValue value)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache.Put(key, value);
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
}

// Tests
public class LRUCacheTests
{
    [Fact]
    public void Cache_EvictsLeastRecentlyUsed()
    {
        var cache = new LRUCache<int, string>(2);

        cache.Put(1, "one");
        cache.Put(2, "two");
        cache.Get(1);        // Access 1, making 2 the LRU
        cache.Put(3, "three"); // Evicts 2

        Assert.Null(cache.Get(2));  // 2 was evicted
        Assert.Equal("one", cache.Get(1));
        Assert.Equal("three", cache.Get(3));
    }

    [Fact]
    public void Cache_UpdatesExistingKey()
    {
        var cache = new LRUCache<int, string>(2);

        cache.Put(1, "one");
        cache.Put(1, "ONE");

        Assert.Equal("ONE", cache.Get(1));
        Assert.Equal(1, cache.Count);
    }
}
```

### Communication Tactics
🎯 **Structure your answer**: "The key insight is that we need O(1) for both lookup AND maintaining order. I'll use a dictionary plus a doubly linked list..."

💡 **Emphasize**: Why a doubly linked list - we need to remove nodes from the middle in O(1), which requires knowing both previous and next pointers.

⚠️ **Avoid**: Using a regular List<T> - removing from the middle is O(n). Also consider mentioning real-world alternatives like `MemoryCache` in production.

---

## Question 5: Valid Palindrome

### The Question
> "Write a function to determine if a string is a valid palindrome, considering only alphanumeric characters and ignoring case. For example, 'A man, a plan, a canal: Panama' is a palindrome. Handle edge cases and optimize for space."

### Key Points to Cover
- Two-pointer technique
- Character filtering (alphanumeric only)
- Case normalization
- Edge cases (empty string, single character, all non-alphanumeric)
- Space optimization (in-place vs creating new string)

### Detailed Answer

**Approaches:**
1. **Clean then compare**: Filter string, reverse, compare - O(n) space
2. **Two pointers**: Compare in-place, skip non-alphanumeric - O(1) space

The two-pointer approach is more space-efficient and demonstrates good algorithm design.

### Code Example
```csharp
public class PalindromeSolution
{
    // Approach 1: Clean and reverse (O(n) space)
    public bool IsPalindrome_Simple(string s)
    {
        if (string.IsNullOrEmpty(s)) return true;

        var cleaned = new string(s
            .Where(char.IsLetterOrDigit)
            .Select(char.ToLower)
            .ToArray());

        var reversed = new string(cleaned.Reverse().ToArray());

        return cleaned == reversed;
    }

    // Approach 2: Two pointers (O(1) space) - PREFERRED
    public bool IsPalindrome(string s)
    {
        if (string.IsNullOrEmpty(s)) return true;

        int left = 0;
        int right = s.Length - 1;

        while (left < right)
        {
            // Skip non-alphanumeric from left
            while (left < right && !char.IsLetterOrDigit(s[left]))
            {
                left++;
            }

            // Skip non-alphanumeric from right
            while (left < right && !char.IsLetterOrDigit(s[right]))
            {
                right--;
            }

            // Compare characters (case-insensitive)
            if (char.ToLower(s[left]) != char.ToLower(s[right]))
            {
                return false;
            }

            left++;
            right--;
        }

        return true;
    }

    // Using Span<char> for performance (modern C#)
    public bool IsPalindrome_Span(string s)
    {
        if (string.IsNullOrEmpty(s)) return true;

        ReadOnlySpan<char> span = s.AsSpan();
        int left = 0;
        int right = span.Length - 1;

        while (left < right)
        {
            while (left < right && !char.IsLetterOrDigit(span[left]))
                left++;

            while (left < right && !char.IsLetterOrDigit(span[right]))
                right--;

            if (char.ToLower(span[left]) != char.ToLower(span[right]))
                return false;

            left++;
            right--;
        }

        return true;
    }

    // Bonus: Check if palindrome can be formed by removing at most one character
    public bool ValidPalindromeII(string s)
    {
        int left = 0;
        int right = s.Length - 1;

        while (left < right)
        {
            if (s[left] != s[right])
            {
                // Try removing either left or right character
                return IsPalindromeRange(s, left + 1, right) ||
                       IsPalindromeRange(s, left, right - 1);
            }
            left++;
            right--;
        }

        return true;
    }

    private bool IsPalindromeRange(string s, int left, int right)
    {
        while (left < right)
        {
            if (s[left] != s[right]) return false;
            left++;
            right--;
        }
        return true;
    }
}

// Comprehensive tests
public class PalindromeTests
{
    private readonly PalindromeSolution _solution = new();

    [Theory]
    [InlineData("A man, a plan, a canal: Panama", true)]
    [InlineData("race a car", false)]
    [InlineData("", true)]
    [InlineData(" ", true)]
    [InlineData("a", true)]
    [InlineData(".,", true)]  // All non-alphanumeric
    [InlineData("0P", false)] // Mixed alphanumeric
    [InlineData("abba", true)]
    [InlineData("abcba", true)]
    [InlineData("Aa", true)]  // Case insensitive
    public void IsPalindrome_HandlesAllCases(string input, bool expected)
    {
        Assert.Equal(expected, _solution.IsPalindrome(input));
        Assert.Equal(expected, _solution.IsPalindrome_Simple(input));
        Assert.Equal(expected, _solution.IsPalindrome_Span(input));
    }

    [Fact]
    public void IsPalindrome_PerformanceComparison()
    {
        var longPalindrome = new string('a', 100000) + new string('b', 100000) +
                             new string('a', 100000);

        // All approaches should handle long strings
        var sw = System.Diagnostics.Stopwatch.StartNew();
        _solution.IsPalindrome(longPalindrome);
        var twoPointerTime = sw.ElapsedMilliseconds;

        sw.Restart();
        _solution.IsPalindrome_Simple(longPalindrome);
        var simpleTime = sw.ElapsedMilliseconds;

        // Two-pointer should be faster due to no allocations
        // (In practice, difference may be small for modern GC)
    }
}
```

### Communication Tactics
🎯 **Structure your answer**: "I'll use two pointers starting from both ends. This gives O(n) time and O(1) space. Let me handle the edge cases first..."

💡 **Emphasize**: Why two pointers is better than creating a new cleaned string - we avoid allocations and work in-place. Mention Span<T> for bonus points.

⚠️ **Avoid**: Forgetting edge cases like empty strings, all non-alphanumeric characters, or strings with only one character. Also be careful with the while loop conditions to prevent index out of bounds.

---

## Coding Interview Tips Summary

### Before Coding
1. **Clarify the problem** - Ask about edge cases, constraints, expected input size
2. **Think out loud** - Share your thought process
3. **Start with brute force** - Then optimize

### While Coding
1. **Write clean code** - Good variable names, proper structure
2. **Handle edge cases** - Empty input, single element, duplicates
3. **Test your code** - Walk through with examples

### Complexity Analysis
| Algorithm | Time | Space | Use Case |
|-----------|------|-------|----------|
| Two Sum (Hash) | O(n) | O(n) | Trading space for time |
| Sliding Window | O(n) | O(k) | Contiguous subarray/substring |
| Two Pointers | O(n) | O(1) | Sorted arrays, palindromes |
| Binary Search | O(log n) | O(1) | Sorted data, search space reduction |

### Common Patterns
- **Hash Map**: Fast lookup, counting, grouping
- **Two Pointers**: Sorted arrays, finding pairs
- **Sliding Window**: Subarray/substring problems
- **Stack**: Matching brackets, monotonic sequences
- **BFS/DFS**: Graph/tree traversal
