# Coding Practice

> LINQ fluency, algorithm patterns, and practice resources

---

## LINQ Mastery

LINQ fluency is essential for .NET interviews. Know these operations well.

### Query Syntax vs Method Syntax

```csharp
// Query syntax (SQL-like)
var query = from o in orders
            where o.Status == OrderStatus.Pending
            orderby o.CreatedAt descending
            select new { o.Id, o.Total };

// Method syntax (more common in production code)
var query = orders
    .Where(o => o.Status == OrderStatus.Pending)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new { o.Id, o.Total });
```

### Essential LINQ Operations

#### Filtering

```csharp
// Where - filter elements
var pendingOrders = orders.Where(o => o.Status == OrderStatus.Pending);

// OfType - filter by type
var specialOrders = orders.OfType<ExpressOrder>();

// Distinct - remove duplicates
var uniqueCategories = products.Select(p => p.Category).Distinct();

// DistinctBy (.NET 6+) - distinct by key
var uniqueByCategory = products.DistinctBy(p => p.Category);
```

#### Projection

```csharp
// Select - transform elements
var names = customers.Select(c => c.Name);
var dtos = orders.Select(o => new OrderDto { Id = o.Id, Total = o.Total });

// SelectMany - flatten nested collections
var allItems = orders.SelectMany(o => o.Items);

// Example: Get all tags from all posts
var tags = posts.SelectMany(p => p.Tags);
```

#### Ordering

```csharp
// OrderBy / OrderByDescending
var sorted = products.OrderBy(p => p.Name);
var newest = orders.OrderByDescending(o => o.CreatedAt);

// ThenBy / ThenByDescending - secondary sort
var sorted = products
    .OrderBy(p => p.Category)
    .ThenByDescending(p => p.Price);
```

#### Aggregation

```csharp
// Count
var totalOrders = orders.Count();
var pendingCount = orders.Count(o => o.Status == OrderStatus.Pending);

// Sum
var totalRevenue = orders.Sum(o => o.Total);

// Average
var avgOrderValue = orders.Average(o => o.Total);

// Min / Max
var cheapest = products.Min(p => p.Price);
var mostExpensive = products.Max(p => p.Price);

// MinBy / MaxBy (.NET 6+) - return the element, not the value
var cheapestProduct = products.MinBy(p => p.Price);
var mostExpensiveProduct = products.MaxBy(p => p.Price);

// Aggregate - custom aggregation
var totalWithTax = orders.Aggregate(0m, (sum, o) => sum + o.Total * 1.18m);

// Product of numbers
var product = numbers.Aggregate(1, (acc, n) => acc * n);
```

#### Grouping

```csharp
// GroupBy
var ordersByStatus = orders.GroupBy(o => o.Status);

foreach (var group in ordersByStatus)
{
    Console.WriteLine($"Status: {group.Key}, Count: {group.Count()}");
}

// GroupBy with result selector
var summary = orders.GroupBy(
    o => o.Status,
    (status, ordersInGroup) => new
    {
        Status = status,
        Count = ordersInGroup.Count(),
        TotalRevenue = ordersInGroup.Sum(o => o.Total)
    });

// Multiple keys
var byYearAndMonth = orders.GroupBy(o => new
{
    Year = o.CreatedAt.Year,
    Month = o.CreatedAt.Month
});
```

#### Joining

```csharp
// Join - inner join
var orderCustomers = orders.Join(
    customers,
    o => o.CustomerId,
    c => c.Id,
    (order, customer) => new
    {
        OrderId = order.Id,
        CustomerName = customer.Name,
        Total = order.Total
    });

// GroupJoin - left outer join
var customersWithOrders = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (customer, customerOrders) => new
    {
        Customer = customer.Name,
        Orders = customerOrders.ToList(),
        TotalSpent = customerOrders.Sum(o => o.Total)
    });

// Zip - combine two sequences
var pairs = numbers.Zip(letters, (n, l) => $"{n}{l}");
// [1,2,3] + [a,b,c] = ["1a", "2b", "3c"]
```

#### Set Operations

```csharp
// Union - combine distinct
var allProducts = storeA.Union(storeB);

// Intersect - common elements
var commonProducts = storeA.Intersect(storeB);

// Except - elements in first but not second
var exclusiveToA = storeA.Except(storeB);

// With comparer
var common = listA.Intersect(listB, StringComparer.OrdinalIgnoreCase);
```

#### Element Operations

```csharp
// First / FirstOrDefault
var first = orders.First();  // Throws if empty
var firstOrNull = orders.FirstOrDefault();  // null if empty
var firstPending = orders.First(o => o.Status == OrderStatus.Pending);

// Single / SingleOrDefault
var order = orders.Single(o => o.Id == orderId);  // Throws if not exactly one
var orderOrNull = orders.SingleOrDefault(o => o.Id == orderId);

// Last / LastOrDefault
var latest = orders.OrderBy(o => o.CreatedAt).Last();

// ElementAt / ElementAtOrDefault
var third = orders.ElementAt(2);
var thirdOrNull = orders.ElementAtOrDefault(2);
```

#### Quantifiers

```csharp
// Any - at least one matches
bool hasPending = orders.Any(o => o.Status == OrderStatus.Pending);
bool hasOrders = orders.Any();  // Non-empty check

// All - all elements match
bool allShipped = orders.All(o => o.Status == OrderStatus.Shipped);

// Contains
bool hasProductX = productIds.Contains("PROD-001");
```

#### Partitioning

```csharp
// Take - first n elements
var topFive = orders.OrderByDescending(o => o.Total).Take(5);

// TakeLast - last n elements
var lastThree = orders.TakeLast(3);

// Skip - skip first n elements
var afterFirst = orders.Skip(10);

// SkipLast - skip last n elements
var withoutLast = orders.SkipLast(1);

// TakeWhile / SkipWhile
var beforeExpensive = products.OrderBy(p => p.Price)
    .TakeWhile(p => p.Price < 1000);

// Chunk (.NET 6+) - split into chunks
var batches = items.Chunk(100);  // IEnumerable<Item[]> with 100 items each
```

### Advanced LINQ Patterns

```csharp
// Index with Select
var indexed = items.Select((item, index) => new { Index = index, Item = item });

// Index (.NET 9)
foreach (var (index, item) in items.Index())
{
    Console.WriteLine($"{index}: {item}");
}

// Running total
var runningTotal = orders
    .OrderBy(o => o.CreatedAt)
    .Select((o, i) => new
    {
        Order = o,
        RunningTotal = orders.Take(i + 1).Sum(x => x.Total)
    });

// Flatten and group
var productSales = orders
    .SelectMany(o => o.Items, (order, item) => new { order, item })
    .GroupBy(x => x.item.ProductId)
    .Select(g => new
    {
        ProductId = g.Key,
        TotalSold = g.Sum(x => x.item.Quantity),
        Revenue = g.Sum(x => x.item.Quantity * x.item.Price)
    });

// Pagination
var page = orders
    .OrderByDescending(o => o.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize);
```

---

## Common Algorithm Patterns

### Two Pointers

```csharp
// Problem: Find pair with target sum in sorted array
public int[] TwoSum(int[] nums, int target)
{
    int left = 0, right = nums.Length - 1;

    while (left < right)
    {
        int sum = nums[left] + nums[right];

        if (sum == target)
            return new[] { left, right };
        else if (sum < target)
            left++;
        else
            right--;
    }

    return Array.Empty<int>();
}

// Problem: Remove duplicates from sorted array in-place
public int RemoveDuplicates(int[] nums)
{
    if (nums.Length == 0) return 0;

    int writeIndex = 1;

    for (int i = 1; i < nums.Length; i++)
    {
        if (nums[i] != nums[i - 1])
        {
            nums[writeIndex] = nums[i];
            writeIndex++;
        }
    }

    return writeIndex;
}

// Problem: Is palindrome?
public bool IsPalindrome(string s)
{
    int left = 0, right = s.Length - 1;

    while (left < right)
    {
        // Skip non-alphanumeric
        while (left < right && !char.IsLetterOrDigit(s[left]))
            left++;
        while (left < right && !char.IsLetterOrDigit(s[right]))
            right--;

        if (char.ToLower(s[left]) != char.ToLower(s[right]))
            return false;

        left++;
        right--;
    }

    return true;
}
```

### Sliding Window

```csharp
// Problem: Maximum sum of subarray of size k
public int MaxSumSubarray(int[] nums, int k)
{
    int windowSum = 0;
    int maxSum = int.MinValue;

    for (int i = 0; i < nums.Length; i++)
    {
        windowSum += nums[i];

        if (i >= k - 1)
        {
            maxSum = Math.Max(maxSum, windowSum);
            windowSum -= nums[i - (k - 1)];
        }
    }

    return maxSum;
}

// Problem: Longest substring without repeating characters
public int LengthOfLongestSubstring(string s)
{
    var seen = new HashSet<char>();
    int left = 0, maxLength = 0;

    for (int right = 0; right < s.Length; right++)
    {
        while (seen.Contains(s[right]))
        {
            seen.Remove(s[left]);
            left++;
        }

        seen.Add(s[right]);
        maxLength = Math.Max(maxLength, right - left + 1);
    }

    return maxLength;
}
```

### Hash Maps for Frequency/Lookup

```csharp
// Problem: Two sum (unsorted) - O(n) with hash map
public int[] TwoSumUnsorted(int[] nums, int target)
{
    var seen = new Dictionary<int, int>();

    for (int i = 0; i < nums.Length; i++)
    {
        int complement = target - nums[i];

        if (seen.TryGetValue(complement, out int complementIndex))
            return new[] { complementIndex, i };

        seen[nums[i]] = i;
    }

    return Array.Empty<int>();
}

// Problem: Find first non-repeating character
public char FirstUniqChar(string s)
{
    var frequency = new Dictionary<char, int>();

    foreach (char c in s)
    {
        frequency[c] = frequency.GetValueOrDefault(c, 0) + 1;
    }

    foreach (char c in s)
    {
        if (frequency[c] == 1)
            return c;
    }

    return '\0';
}

// Problem: Group anagrams
public IList<IList<string>> GroupAnagrams(string[] strs)
{
    var groups = new Dictionary<string, List<string>>();

    foreach (var s in strs)
    {
        // Key: sorted characters
        var key = new string(s.OrderBy(c => c).ToArray());

        if (!groups.ContainsKey(key))
            groups[key] = new List<string>();

        groups[key].Add(s);
    }

    return groups.Values.ToList<IList<string>>();
}
```

### Stack Problems

```csharp
// Problem: Valid parentheses
public bool IsValid(string s)
{
    var stack = new Stack<char>();
    var pairs = new Dictionary<char, char>
    {
        { ')', '(' },
        { ']', '[' },
        { '}', '{' }
    };

    foreach (char c in s)
    {
        if (pairs.ContainsValue(c))
        {
            stack.Push(c);
        }
        else if (pairs.ContainsKey(c))
        {
            if (stack.Count == 0 || stack.Pop() != pairs[c])
                return false;
        }
    }

    return stack.Count == 0;
}

// Problem: Daily temperatures (next greater element)
public int[] DailyTemperatures(int[] temperatures)
{
    var result = new int[temperatures.Length];
    var stack = new Stack<int>();  // Stack of indices

    for (int i = 0; i < temperatures.Length; i++)
    {
        while (stack.Count > 0 && temperatures[i] > temperatures[stack.Peek()])
        {
            int prevIndex = stack.Pop();
            result[prevIndex] = i - prevIndex;
        }
        stack.Push(i);
    }

    return result;
}
```

### String Manipulation

```csharp
// Problem: Reverse words in string
public string ReverseWords(string s)
{
    return string.Join(" ", s.Split(' ', StringSplitOptions.RemoveEmptyEntries).Reverse());
}

// Problem: Check if strings are rotations
public bool IsRotation(string s1, string s2)
{
    if (s1.Length != s2.Length) return false;
    return (s1 + s1).Contains(s2);
}

// Problem: Longest common prefix
public string LongestCommonPrefix(string[] strs)
{
    if (strs.Length == 0) return "";

    var prefix = new StringBuilder();
    var first = strs[0];

    for (int i = 0; i < first.Length; i++)
    {
        char c = first[i];

        if (strs.All(s => i < s.Length && s[i] == c))
            prefix.Append(c);
        else
            break;
    }

    return prefix.ToString();
}

// Problem: String compression
public string Compress(string s)
{
    var sb = new StringBuilder();
    int count = 1;

    for (int i = 1; i <= s.Length; i++)
    {
        if (i < s.Length && s[i] == s[i - 1])
        {
            count++;
        }
        else
        {
            sb.Append(s[i - 1]);
            if (count > 1) sb.Append(count);
            count = 1;
        }
    }

    return sb.Length < s.Length ? sb.ToString() : s;
}
```

### Binary Search

```csharp
// Standard binary search
public int BinarySearch(int[] nums, int target)
{
    int left = 0, right = nums.Length - 1;

    while (left <= right)
    {
        int mid = left + (right - left) / 2;  // Avoid overflow

        if (nums[mid] == target)
            return mid;
        else if (nums[mid] < target)
            left = mid + 1;
        else
            right = mid - 1;
    }

    return -1;  // Not found
}

// Find first occurrence
public int FindFirst(int[] nums, int target)
{
    int left = 0, right = nums.Length - 1;
    int result = -1;

    while (left <= right)
    {
        int mid = left + (right - left) / 2;

        if (nums[mid] == target)
        {
            result = mid;
            right = mid - 1;  // Continue searching left
        }
        else if (nums[mid] < target)
            left = mid + 1;
        else
            right = mid - 1;
    }

    return result;
}
```

---

## Common Interview Coding Questions

### FizzBuzz (Classic)

```csharp
public IEnumerable<string> FizzBuzz(int n)
{
    for (int i = 1; i <= n; i++)
    {
        yield return (i % 3 == 0, i % 5 == 0) switch
        {
            (true, true) => "FizzBuzz",
            (true, false) => "Fizz",
            (false, true) => "Buzz",
            _ => i.ToString()
        };
    }
}

// Or with LINQ
public IEnumerable<string> FizzBuzzLinq(int n) =>
    Enumerable.Range(1, n).Select(i =>
        (i % 15 == 0) ? "FizzBuzz" :
        (i % 3 == 0) ? "Fizz" :
        (i % 5 == 0) ? "Buzz" :
        i.ToString());
```

### Fibonacci

```csharp
// Iterative (efficient)
public int Fibonacci(int n)
{
    if (n <= 1) return n;

    int prev = 0, curr = 1;

    for (int i = 2; i <= n; i++)
    {
        int next = prev + curr;
        prev = curr;
        curr = next;
    }

    return curr;
}

// With memoization
public int FibMemo(int n, Dictionary<int, int> memo = null)
{
    memo ??= new Dictionary<int, int>();

    if (n <= 1) return n;
    if (memo.ContainsKey(n)) return memo[n];

    memo[n] = FibMemo(n - 1, memo) + FibMemo(n - 2, memo);
    return memo[n];
}
```

### Merge Two Sorted Arrays

```csharp
public int[] MergeSorted(int[] a, int[] b)
{
    var result = new int[a.Length + b.Length];
    int i = 0, j = 0, k = 0;

    while (i < a.Length && j < b.Length)
    {
        if (a[i] <= b[j])
            result[k++] = a[i++];
        else
            result[k++] = b[j++];
    }

    while (i < a.Length) result[k++] = a[i++];
    while (j < b.Length) result[k++] = b[j++];

    return result;
}
```

### Find Missing Number

```csharp
// Array of 0 to n with one missing
public int FindMissing(int[] nums)
{
    int n = nums.Length;
    int expectedSum = n * (n + 1) / 2;
    int actualSum = nums.Sum();
    return expectedSum - actualSum;
}

// Using XOR (no overflow risk)
public int FindMissingXor(int[] nums)
{
    int xor = nums.Length;

    for (int i = 0; i < nums.Length; i++)
    {
        xor ^= i ^ nums[i];
    }

    return xor;
}
```

---

## Practice Resources

### Platforms

| Platform | Best For |
|----------|----------|
| **LeetCode** | Most popular, good problem variety, company-specific lists |
| **HackerRank** | Good for beginners, certification tests |
| **Codewars** | Fun, ranking system (kyu), good for daily practice |
| **Exercism** | Mentored learning, many languages |

### Recommended LeetCode Problems

**Easy (Warm-up):**
- Two Sum
- Valid Parentheses
- Merge Two Sorted Lists
- Best Time to Buy/Sell Stock
- Valid Palindrome
- Reverse String
- Contains Duplicate

**Medium (Interview Level):**
- 3Sum
- Longest Substring Without Repeating
- Group Anagrams
- Product of Array Except Self
- Top K Frequent Elements
- Validate BST
- LRU Cache

### Daily Practice Plan

```
Week 1-2: Arrays & Strings
- Two pointers
- Sliding window
- String manipulation

Week 3-4: Hash Tables & Sets
- Frequency counting
- Lookup optimization
- Anagrams, duplicates

Week 5-6: Stacks & Queues
- Valid parentheses
- Next greater element
- Queue with stacks

Week 7-8: Linked Lists
- Reverse
- Cycle detection
- Merge sorted

Week 9-10: Trees
- Traversals
- BST operations
- Tree depth/height

Week 11-12: Review & Mock Interviews
- Practice under time pressure
- Explain while coding
```

---

## Interview Tips

### Before Coding

1. **Clarify the problem** - Ask about edge cases, constraints
2. **Think aloud** - Explain your approach before coding
3. **Start with brute force** - Then optimize

### While Coding

1. **Use meaningful names** - Not `i, j, k` for everything
2. **Handle edge cases** - Empty input, single element
3. **Test your code** - Walk through with example

### Common Mistakes to Avoid

```csharp
// ❌ Off-by-one errors
for (int i = 0; i <= arr.Length; i++)  // ArrayIndexOutOfBounds!

// ✅ Correct
for (int i = 0; i < arr.Length; i++)

// ❌ Modifying collection while iterating
foreach (var item in list)
{
    if (condition) list.Remove(item);  // InvalidOperationException!
}

// ✅ Use ToList() or iterate backwards
foreach (var item in list.ToList())
{
    if (condition) list.Remove(item);
}

// ❌ Integer overflow
int mid = (left + right) / 2;  // Can overflow!

// ✅ Safe calculation
int mid = left + (right - left) / 2;

// ❌ Not handling null
string.Length  // NullReferenceException!

// ✅ Null check
string?.Length ?? 0
```
