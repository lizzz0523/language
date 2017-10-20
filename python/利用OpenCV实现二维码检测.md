
# 利用OpenCV实现二维码检测


```python
import numpy as np
import matplotlib.pyplot as plt
import cv2
```

辅助函数，用于在jupyter里显示图片


```python
def imshow_rgb(title, img):
    plt.title(title)
    plt.imshow(cv2.cvtColor(img, cv2.COLOR_BGR2RGB))
    plt.show()
    
def imshow_gs(title, img):
    plt.title(title)
    plt.imshow(img, cmap='gray')
    plt.show()
```

读入图片，并显示


```python
img = cv2.imread('timg4.jpg')
imshow_rgb('original', img)
```


![png](output_5_0.png)


把图片转换成灰度图


```python
img_gs = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
imshow_gs('grayscale', img_gs)
```


![png](output_7_0.png)


对图片进行常规性的裁剪，只保留比较靠近中心区域的图像


```python
roi = img[10:-10,10:-10]
imshow_rgb('roi_rgb', roi)

roi_gs = img_gs[10:-10,10:-10]
imshow_gs('roi_gs', roi_gs)
```


![png](output_9_0.png)



![png](output_9_1.png)


保存二值化图像


```python
_, roi_thresh = cv2.threshold(roi_gs, 120, 255, cv2.THRESH_BINARY)
imshow_gs('roi_thresh', roi_thresh)
```


![png](output_11_0.png)


对图片进行高斯模糊，去掉高频信息，方便后面的边缘提取


```python
roi_blur = cv2.GaussianBlur(roi_gs, (3, 3), 1)
imshow_gs('gaussian', roi_blur)
```


![png](output_13_0.png)


对图片进行边缘提取


```python
roi_edge = cv2.Canny(roi_blur, 100, 100)
imshow_gs('edage', roi_edge)
```


![png](output_15_0.png)


根据边缘信息，提取图像中的轮廓


```python
_, contours, hierarchy = cv2.findContours(roi_edge, cv2.RETR_TREE,cv2.CHAIN_APPROX_SIMPLE)

roi_tmp = roi.copy()

for i in range(len(contours)):
    cv2.drawContours(roi_tmp, contours, i, (0, 0, 255), 3)
    
imshow_rgb('check', roi_tmp)
```


![png](output_17_0.png)


根据轮廓的等级信息，找到二维码的Position Detection Pattern


```python
hierarchy = hierarchy[0]
found = []

for i in range(len(contours)):
    k = i
    c = 0
    
    while hierarchy[k][2] != -1:
        k = hierarchy[k][2]
        c += 1
        
    if c >= 5:
        found.append(i)

print(found)
```

    [315, 381, 422]
    

把找到的Position Detection Pattern描绘出来


```python
roi_tmp = roi.copy()

for i in found:
    cv2.drawContours(roi_tmp, contours, i, (0, 255, 0), 3)

imshow_rgb('check', roi_tmp)
```


![png](output_21_0.png)


但有时候，找到的Position Detection Pattern会超过三个，也就是有一些假的Position Detection Pattern混进了我们的结果中，需要进一步排除
这是我们可以利用，二维码的Timing Pattern

首先把所有待选轮廓的四个顶点找到


```python
boxes = []

for i in found:
    rect = cv2.minAreaRect(contours[i])
    box = cv2.boxPoints(rect)
    box = np.int0(box)
    boxes.append(box)
    
print(boxes)
```

    [array([[337, 369],
           [294, 353],
           [310, 310],
           [353, 326]], dtype=int64), array([[521, 288],
           [477, 271],
           [493, 228],
           [537, 244]], dtype=int64), array([[386, 236],
           [344, 220],
           [360, 178],
           [402, 194]], dtype=int64)]
    

接下来，我们就要开始计算这些顶点之间的距离，这里我们定义了两个函数，一个是计算两个点之间的距离，一个是用于查找两个轮廓之间，距离最短的两组顶点

在寻找到距离最短的两组顶点后，我们需要顶点坐标进行校正，已使其对准二维码方块的中心
![fixed](./fixed.jpg)


```python
import math

def distance(pt1, pt2):
    return math.sqrt(np.sum((pt1 - pt2) ** 2))

def find_shortest(box1, box2):
    t1 = t2 = None
    d1 = d2 = np.iinfo('i').max
    
    for pt1 in box1:
        for pt2 in box2:
            d = distance(pt1, pt2)
            t = (pt1, pt2)
            
            if d < d2:
                if d < d1:
                    t1, t2 = t, t1
                    d1, d2 = d, d1
                else:
                    t2 = t
                    d2 = d
                    
    pt1, pt2 = t1
    pt3, pt4 = t2
    
    # 这里是为了校正坐标到放开中心
    r1 = (pt1 - pt3) // 12
    r2 = (pt2 - pt4) // 12
    
    pt1 = pt1 - r1
    pt3 = pt3 + r1
    pt2 = pt2 - r2
    pt4 = pt4 + r2
    
    return pt1, pt2, pt3, pt4
```


```python
pts = find_shortest(boxes[1], boxes[2])

roi_tmp = roi.copy()

for pt in map(tuple, pts):
    cv2.circle(roi_tmp, pt, 3, (0, 0, 255), 3)
    
imshow_rgb('points', roi_tmp)
```


![png](output_28_0.png)


接下来，我们就要看看这两种顶点之间的连线，是否符合Timing Pattern

由于Timing Pattern是黑白相间的，我们通过对连线上的黑白像素点进行统计，然后计算其方差，如果方差较少，即黑白点是均匀分布的


```python
def check_timing(img, pt1, pt2):
    step = 20
    rate = (pt1 - pt2) // step
    line = []
    
    if np.sum(rate) == 0:
        return False
    
    for i in range(step):
        x, y = pt2 + i * rate
        v = img[y, x]
        line.append(v)

    start = 0
    stop = len(line)
    
    while start < stop and line[start] == 0:
        start += 1
        
    while start < stop and line[stop - 1] == 0:
        stop -= 1
        
    if stop <= start:
        return False
            
    last = line[start]
    count = 1
    counts = []
    
    for i in range(start + 1, stop):
        if line[i] == last:
            count += 1
        else:
            counts.append(count)
            count = 1
            last = line[i]
            
    print(counts)
            
    if len(counts) < 4:
        return False
    
    print(np.var(counts))
    
    return np.var(counts) < 50


check_timing(roi_thresh, pts[0], pts[1])
```

    [1, 2, 3, 3]
    0.6875
    




    True



根据是否拥有Timing Pattern，我们就可以对所有轮廓进行筛选，最终找到我们要的Position Detection Pattern


```python
target = set()

for i in range(len(boxes)):
    for j in range(i + 1, len(boxes)):
        pts = find_shortest(boxes[1], boxes[2])
        r1 = check_timing(roi_thresh, pts[0], pts[1])
        r2 = check_timing(roi_thresh, pts[2], pts[3])
        
        if r1 or r2:
            target.add(i)
            target.add(j)
            
print(target)
```

    [1, 2, 3, 3]
    0.6875
    [1, 2, 3, 2, 4, 2]
    0.888888888889
    [1, 2, 3, 3]
    0.6875
    [1, 2, 3, 2, 4, 2]
    0.888888888889
    [1, 2, 3, 3]
    0.6875
    [1, 2, 3, 2, 4, 2]
    0.888888888889
    {0, 1, 2}
    

然后我们就根据这些Position Detection Pattern的坐标，找到二维码的四个顶点


```python
contours_all = []

for i in target:
    contour = contours[found[i]]
    
    for sub_contour in contour:
        for pt in sub_contour:
            contours_all.append(pt)

contours_all = np.expand_dims(contours_all, axis=0)

rect = cv2.minAreaRect(contours_all)
box = cv2.boxPoints(rect)
box = np.int0(box)

box
```




    array([[471, 419],
           [294, 353],
           [360, 178],
           [537, 244]], dtype=int64)




```python
roi_tmp = roi.copy()
cv2.polylines(roi_tmp, np.int32([box]), True, (0, 0, 255), 3)
imshow_rgb('final', roi_tmp)
```


![png](output_36_0.png)


接下来，就要对二维进行几何校正


```python
pt1 = np.float32(box[:3])
pt2 = np.float32([[0,0],[200,0],[200, 200]])

print(pt1)
print(pt2)

M = cv2.getAffineTransform(pt1, pt2)

print(M)
```

    [[ 471.  419.]
     [ 294.  353.]
     [ 360.  178.]]
    [[   0.    0.]
     [ 200.    0.]
     [ 200.  200.]]
    [[ -9.90631457e-01  -3.73609578e-01   6.23129829e+02]
     [  3.73609578e-01  -1.00195296e+00   2.43848179e+02]]
    


```python
roi_tr = cv2.warpAffine(roi_thresh, M, roi.shape[:-1])[:200,:200]
imshow_gs('transform', roi_tr)
```


![png](output_39_0.png)


以上~
