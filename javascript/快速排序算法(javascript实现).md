### 快速排序

```javascript
    function quickSort(arr) {
        function sort(arr, left, right) {
            if (left >= right) {
                return;
            }

            var mid = left;
            var low = left;
            var high = right;

            while (low < high) {
                while (arr[high] >= arr[mid] && low < high) {
                    high--;
                }

                while (arr[low] <= arr[mid] && low < high) {
                    low++;
                }

                swap(arr, low, high);
            }

            swap(arr, low, mid);

            sort(arr, left, low - 1);
            sort(arr, low + 1, right);
        }

        return sort(arr, 0, arr.length - 1);
    }

    function swap(arr, i, j) {
        var temp;

        temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
```