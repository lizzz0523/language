### 桶排序

```javascript
    function bucketSort(arr) {
        var buckets = [];
        var pos = 0;

        for (var i = 0; i < 10; i++) {
            buckets[i] = []l
        }

        for (var j = 0; j <= arr.length - 1; j++) {
            buckets[arr[j] / 10 >> 0].push(arr[j]);
        }

        for (var k = 0; k < 10; k++) {
            quickSort(buckets[k]);
            
            while (buckets[k].length) {
                arr[pos++] = buckets[k].shift();
            }
        }
    }
```