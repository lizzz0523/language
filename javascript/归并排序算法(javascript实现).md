### 归并排序

```javascript
    function mergeSort(arr) {
        function sort(arr, temp, low, high) {
            if (low >= high) {
                return;
            }

            var mid = (high + low) / 2 >> 0; 

            sort(arr, temp, low, mid);
            sort(arr, temp, mid + 1, high);
            merge(arr, temp, low, mid, mid + 1, high);
        }

        function merge(arr, temp, low, lowEnd, high, highEnd) {
            var pos = low;
            var start = low;
            var end = highEnd;

            while (low <= lowEnd && high <= highEnd) {
                if (arr[low] < arr[high]) {
                    temp[pos++] = arr[low++];
                } else {
                    temp[pos++] = arr[high++];
                }
            }

            while (low <= lowEnd) {
                temp[pos++] = arr[low++];
            }

            while (high <= highEnd) {
                temp[pos++] = arr[high++];
            }

            for (var i = start; i <= end; i++) {
                arr[i] = temp[i];
            }
        }

        return sort(arr, [], 0, arr.length - 1);
    }
```