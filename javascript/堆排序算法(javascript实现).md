### 堆排序

```javascript
    function heapSort(arr) {
        build(arr);
        sort(arr);
    }

    function swap(arr, i, j) {
        var temp;

        temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
    
    function sort(arr) {
        var i = arr.length - 1;

        while (i >= 0) {
            swap(arr, 0, i);
            i--;
            shiftdown(arr, 0);
        }
    }

    function build() {
        for (var i = arr.length / 2 >> 0; i >= 0; i--) {
            shiftdown(arr, i);
        }
    }

    function shiftdown() {

    }
```