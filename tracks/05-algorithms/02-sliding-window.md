# 02. Sliding Window

[← Назад к списку тем](README.md)

---

## Теория: Sliding Window паттерн

### Когда использовать
- Задачи на подстроки/подмассивы
- "Найти минимальный/максимальный подмассив с условием X"
- Условие связано с суммой, количеством уникальных элементов, частотой

### Два типа окна
1. **Fixed size:** окно фиксированного размера k
2. **Variable size:** окно расширяется/сужается по условию

### Шаблон

```python
# Variable size window
def sliding_window(arr):
    left = 0
    result = 0
    window_state = ...  # hash map, sum, etc.

    for right in range(len(arr)):
        # Расширяем окно: добавляем arr[right]
        update_window(arr[right])

        # Сужаем окно пока условие нарушено
        while window_is_invalid():
            # Удаляем arr[left]
            remove_from_window(arr[left])
            left += 1

        # Обновляем результат
        result = max(result, right - left + 1)

    return result
```

---

## Задача 1: Maximum Sum Subarray of Size K

### Паттерн
Fixed Size Sliding Window

### Условие
Найти максимальную сумму подмассива размера k.

### Решение

```python
def max_sum_subarray(nums: list[int], k: int) -> int:
    if len(nums) < k:
        return 0

    # Сумма первого окна
    window_sum = sum(nums[:k])
    max_sum = window_sum

    # Сдвигаем окно
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]  # Добавляем новый, убираем старый
        max_sum = max(max_sum, window_sum)

    return max_sum

# Пример
nums = [2, 1, 5, 1, 3, 2]
k = 3
print(max_sum_subarray(nums, k))  # 9 (5 + 1 + 3)
```

```go
func maxSumSubarray(nums []int, k int) int {
    if len(nums) < k {
        return 0
    }

    windowSum := 0
    for i := 0; i < k; i++ {
        windowSum += nums[i]
    }
    maxSum := windowSum

    for i := k; i < len(nums); i++ {
        windowSum += nums[i] - nums[i-k]
        if windowSum > maxSum {
            maxSum = windowSum
        }
    }
    return maxSum
}
```

### Сложность
- Time: O(n)
- Space: O(1)

---

## Задача 2: Longest Substring Without Repeating Characters

### Паттерн
Variable Size Window + Hash Set

### Условие
Найти длину самой длинной подстроки без повторяющихся символов.

### Решение

```python
def length_of_longest_substring(s: str) -> int:
    char_set = set()
    left = 0
    max_length = 0

    for right in range(len(s)):
        # Сужаем окно пока есть дубликат
        while s[right] in char_set:
            char_set.remove(s[left])
            left += 1

        char_set.add(s[right])
        max_length = max(max_length, right - left + 1)

    return max_length

# Оптимизация с hash map (прыжок сразу к нужной позиции)
def length_of_longest_substring_optimized(s: str) -> int:
    char_index = {}  # char -> last index
    left = 0
    max_length = 0

    for right, char in enumerate(s):
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1

        char_index[char] = right
        max_length = max(max_length, right - left + 1)

    return max_length

# Пример
s = "abcabcbb"
print(length_of_longest_substring(s))  # 3 ("abc")
```

```go
func lengthOfLongestSubstring(s string) int {
    charIndex := make(map[rune]int)
    left := 0
    maxLength := 0

    for right, char := range s {
        if idx, ok := charIndex[char]; ok && idx >= left {
            left = idx + 1
        }

        charIndex[char] = right
        if right-left+1 > maxLength {
            maxLength = right - left + 1
        }
    }
    return maxLength
}
```

### Сложность
- Time: O(n)
- Space: O(min(n, m)), где m — размер алфавита

---

## Задача 3: Minimum Window Substring

### Паттерн
Variable Size Window + Character Count

### Условие
Найти минимальную подстроку s, содержащую все символы строки t.

### Интуиция
1. Расширяем окно пока не соберём все символы из t
2. Сужаем окно пока условие выполняется
3. Запоминаем минимум

### Решение

```python
from collections import Counter

def min_window(s: str, t: str) -> str:
    if not t or not s:
        return ""

    # Подсчёт символов в t
    t_count = Counter(t)
    required = len(t_count)  # Количество уникальных символов

    # Текущее окно
    window_count = {}
    formed = 0  # Сколько символов уже собрали в нужном количестве

    left = 0
    min_len = float('inf')
    min_left = 0

    for right in range(len(s)):
        char = s[right]
        window_count[char] = window_count.get(char, 0) + 1

        # Проверяем, собрали ли символ полностью
        if char in t_count and window_count[char] == t_count[char]:
            formed += 1

        # Сужаем окно
        while formed == required:
            # Обновляем результат
            if right - left + 1 < min_len:
                min_len = right - left + 1
                min_left = left

            # Убираем левый символ
            left_char = s[left]
            window_count[left_char] -= 1
            if left_char in t_count and window_count[left_char] < t_count[left_char]:
                formed -= 1
            left += 1

    return "" if min_len == float('inf') else s[min_left:min_left + min_len]

# Пример
s = "ADOBECODEBANC"
t = "ABC"
print(min_window(s, t))  # "BANC"
```

```go
func minWindow(s string, t string) string {
    if len(s) == 0 || len(t) == 0 {
        return ""
    }

    tCount := make(map[byte]int)
    for i := 0; i < len(t); i++ {
        tCount[t[i]]++
    }
    required := len(tCount)

    windowCount := make(map[byte]int)
    formed := 0

    left := 0
    minLen := len(s) + 1
    minLeft := 0

    for right := 0; right < len(s); right++ {
        char := s[right]
        windowCount[char]++

        if cnt, ok := tCount[char]; ok && windowCount[char] == cnt {
            formed++
        }

        for formed == required {
            if right-left+1 < minLen {
                minLen = right - left + 1
                minLeft = left
            }

            leftChar := s[left]
            windowCount[leftChar]--
            if cnt, ok := tCount[leftChar]; ok && windowCount[leftChar] < cnt {
                formed--
            }
            left++
        }
    }

    if minLen == len(s)+1 {
        return ""
    }
    return s[minLeft : minLeft+minLen]
}
```

### Сложность
- Time: O(|s| + |t|)
- Space: O(|s| + |t|)

---

## Задача 4: Longest Repeating Character Replacement

### Паттерн
Variable Window + Max Frequency

### Условие
Дана строка s и число k. Можно заменить до k символов. Найти длину самой длинной подстроки из одинаковых символов.

### Интуиция
Окно валидно если: length - max_freq <= k (можем заменить все символы кроме самого частого)

### Решение

```python
def character_replacement(s: str, k: int) -> int:
    count = {}
    left = 0
    max_freq = 0
    max_length = 0

    for right in range(len(s)):
        count[s[right]] = count.get(s[right], 0) + 1
        max_freq = max(max_freq, count[s[right]])

        # Сужаем если окно невалидно
        # (right - left + 1) - max_freq > k
        while (right - left + 1) - max_freq > k:
            count[s[left]] -= 1
            left += 1

        max_length = max(max_length, right - left + 1)

    return max_length

# Пример
s = "AABABBA"
k = 1
print(character_replacement(s, k))  # 4 ("AABA" или "ABBA")
```

```go
func characterReplacement(s string, k int) int {
    count := make(map[byte]int)
    left := 0
    maxFreq := 0
    maxLength := 0

    for right := 0; right < len(s); right++ {
        count[s[right]]++
        if count[s[right]] > maxFreq {
            maxFreq = count[s[right]]
        }

        for (right-left+1)-maxFreq > k {
            count[s[left]]--
            left++
        }

        if right-left+1 > maxLength {
            maxLength = right - left + 1
        }
    }
    return maxLength
}
```

### Сложность
- Time: O(n)
- Space: O(26) = O(1)

---

## Задача 5: Permutation in String

### Паттерн
Fixed Window + Character Count

### Условие
Проверить, содержит ли s2 перестановку s1.

### Решение

```python
from collections import Counter

def check_inclusion(s1: str, s2: str) -> bool:
    if len(s1) > len(s2):
        return False

    s1_count = Counter(s1)
    window_count = Counter(s2[:len(s1)])

    if s1_count == window_count:
        return True

    for i in range(len(s1), len(s2)):
        # Добавляем новый символ
        window_count[s2[i]] += 1

        # Убираем старый символ
        old_char = s2[i - len(s1)]
        window_count[old_char] -= 1
        if window_count[old_char] == 0:
            del window_count[old_char]

        if s1_count == window_count:
            return True

    return False

# Пример
s1 = "ab"
s2 = "eidbaooo"
print(check_inclusion(s1, s2))  # True ("ba" в позиции 3)
```

```go
func checkInclusion(s1 string, s2 string) bool {
    if len(s1) > len(s2) {
        return false
    }

    var s1Count, windowCount [26]int
    for i := 0; i < len(s1); i++ {
        s1Count[s1[i]-'a']++
        windowCount[s2[i]-'a']++
    }

    if s1Count == windowCount {
        return true
    }

    for i := len(s1); i < len(s2); i++ {
        windowCount[s2[i]-'a']++
        windowCount[s2[i-len(s1)]-'a']--

        if s1Count == windowCount {
            return true
        }
    }
    return false
}
```

### Сложность
- Time: O(n)
- Space: O(1) — фиксированный размер алфавита

---

## Задача 6: Sliding Window Maximum

### Паттерн
Monotonic Deque

### Условие
Для каждого окна размера k найти максимальный элемент.

### Интуиция
Используем deque для хранения индексов в порядке убывания значений. Голова deque — всегда максимум текущего окна.

### Решение

```python
from collections import deque

def max_sliding_window(nums: list[int], k: int) -> list[int]:
    result = []
    dq = deque()  # Хранит индексы

    for i in range(len(nums)):
        # Удаляем элементы вне окна
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        # Удаляем меньшие элементы — они никогда не станут максимумом
        while dq and nums[dq[-1]] < nums[i]:
            dq.pop()

        dq.append(i)

        # Добавляем максимум в результат (начиная с первого полного окна)
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result

# Пример
nums = [1, 3, -1, -3, 5, 3, 6, 7]
k = 3
print(max_sliding_window(nums, k))  # [3, 3, 5, 5, 6, 7]
```

```go
func maxSlidingWindow(nums []int, k int) []int {
    result := []int{}
    dq := []int{} // Индексы

    for i := 0; i < len(nums); i++ {
        // Удаляем вне окна
        for len(dq) > 0 && dq[0] < i-k+1 {
            dq = dq[1:]
        }

        // Удаляем меньшие
        for len(dq) > 0 && nums[dq[len(dq)-1]] < nums[i] {
            dq = dq[:len(dq)-1]
        }

        dq = append(dq, i)

        if i >= k-1 {
            result = append(result, nums[dq[0]])
        }
    }
    return result
}
```

### Сложность
- Time: O(n) — каждый элемент добавляется и удаляется из deque один раз
- Space: O(k)

---

## На интервью

### Как распознать Sliding Window

| Сигнал | Тип окна |
|--------|----------|
| "Подмассив размера k" | Fixed |
| "Минимальный/максимальный подмассив с условием" | Variable |
| "Подстрока содержащая..." | Variable |
| "Не более k различных элементов" | Variable |

### Шаблон ответа

1. Определить тип окна (fixed/variable)
2. Определить что хранить в окне (sum, set, map)
3. Определить условие сужения окна
4. Написать код по шаблону

### Типичные ошибки

1. Путают когда расширять, когда сужать окно
2. Забывают обновить результат после сужения
3. Off-by-one ошибки с границами окна
4. Не обрабатывают случай когда строка короче k

---

[← Назад к списку тем](README.md)
