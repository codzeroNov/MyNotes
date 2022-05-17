**time complexity: O(log n)**

1. when you use *int mid = (left + right)/2*, the mid is biased to the left pointer.
2. count the numbers below the mid, and then move the left or right pointer.
3. for rotated array, get the mid point and check the rotated part, and move pointer towards target.
