> 为嘛写这个，因为手写代码我也很绝望啊

| 排序方法 | 时间复杂度 | 时间复杂度 | 时间复杂度 | 空间复杂度 | 稳定性 |
| :---: | :---: | :---: | :---: | :---: | :---: |
|  | **平均情况** | **最好情况** | **最坏情况** | **辅助存储** |  |
| 冒泡排序 | O(n²) | O(n) | O(n²) | O(1) | 稳定 |
| 选择排序 | O(n²) | O(n²) | O(n²) | O(1) | 不稳定 |
| 快速排序 | O(nlogn) | O(nlogn) | 有序或逆序 O(n²) | O(nlogn) | 不稳定 |
| 直接插入排序 | O(n²) | O(n) | O(n²) | O(1) | 稳定 |
| 折半插入排序 | O(n²) | O(nlogn) | O(n²) | O(1) | 不稳定 |
| 堆排序 | O(nlogn) | O(nlogn) | O(nlogn) | O(1) | 不稳定 |

# 冒泡排序
##### 算法思想
通过与相邻元素的比较和交换，把小的数交换到最前面。

    - (void)sort:(NSMutableArray *)arr {
        //外层循环控制数组大小
        for (int i = 0; i < arr.count; ++i) {
            //内循环从后往前依次比较将小的数据放到最前面
            for (int j = 0; j < arr.count-1-i; ++j) {
                //大元素后移
                if ([arr[j] integerValue]> [arr[j+1] integerValue]) {
                    [arr exchangeObjectAtIndex:j withObjectAtIndex:j+1];
                }
            }
        }
        NSLog(@"冒泡排序结果：%@", arr);
    }

# 选择排序
##### 算法思想
每一趟从前往后查找出值最小的索引（下标），最后通过比较是否需要交换。每一趟都将最小的元素交换到最前面。

    - (void)sort:(NSMutableArray *)arr {    
        for (int i = 0; i < arr.count; i++) {
            for (int j = i+1; j < arr.count; j++) {
                // 每一趟将最小的元素放到最前
                if ([arr[j] integerValue] < [arr[i] integerValue]) {
                    [arr exchangeObjectAtIndex:i withObjectAtIndex:j];
                }
            }
        }
        NSLog(@"选择排序结果：%@", arr);
    }
> 注意：冒泡排序是从后往前扫，使大的往下沉，而小的往上浮；选择排序是从前往后扫，每趟找出值最小的索引，使每趟最小值都交换到该趟的最前面，从而得到升序序列。

# 快速排序
##### 算法思想
每一趟都保证左边比基准小，右边比基准大，而且递归划分排序。 
1. 设置两个变量left、right，排序开始的时候：left=0，right=N-1；
2. 以第一个数组元素作为基准数据，赋值给key，即key=A[0]；
3. 从right开始向前搜索，即由后开始向前搜索(right--)，找到第一个小于key的值A[right]，将A[right]和A[left]互换；
4. 从left开始向后搜索，即由前开始向后搜索(left++)，找到第一个大于key的A[left]，将A[left]和A[right]互换；
5. 重复第3、4步，直到left=right； 
  - 没找到符合条件的值，即3中A[right]不小于key，4中A[left]不大于key的时候改变right、left的值，使得right=right-1，left=left+1，直至找到为止。
  - 找到符合条件的值，进行交换的时候left、right指针位置不变。
  - 另外，left==right这一过程一定正好是left++或right--完成的时候，此时令循环结束

    
    - (void)quickSort:(NSMutableArray *)arr leftIndex:(int)left rightIndex:(int)right {
        if (left < right) {
            int temp = [self getMiddleIndex:arr leftIndex:left rightIndex:right];
            [self quickSort:arr leftIndex:left rightIndex:temp - 1];
            [self quickSort:arr leftIndex:temp + 1 rightIndex:right];
        }
    }
    - (int)getMiddleIndex:(NSMutableArray *)arr leftIndex:(int)left rightIndex:(int)right {
        int tempValue = [arr[left] integerValue];
        while (left < right) {
            while (left < right && tempValue <= [arr[right] integerValue]) {
                right --;
            }
            if (left < right) {
                arr[left] = arr[right];
            }
            while (left < right && [arr[left] integerValue] <= tempValue) {
                left ++;
            }
            if (left < right) {
                arr[right] = arr[left];
            }
        }
        arr[left] = [NSNumber numberWithInt:tempValue];
        return left;
    }

# 插入排序
### 直接插入排序
##### 算法思想
第一次从R[0]~R[n-1]中选取最小值，与R[0]交换，第二次从R[1]~R[n-1]中选取最小值，与R[1]交换，....，第i次从R[i-1]~R[n-1]中选取最小值，与R[i-1]交换，.....，第n-1次从R[n-2]~R[n-1]中选取最小值，与R[n-2]交换，总共通过n-1次，得到一个按排序码从小到大排列的有序序列.

第一个元素就认为是有序的
取第二个元素，判断是否大于第一个元素。若是大于，表示已经有序，不用移动，否则将已经有序的序列整体向后移动一个位置
依此类推，直到所有元素已经有序。

    - (void)sort:(NSMutableArray *)arr {
        // 外层循环用于跑多少趟
        for (int i = 1; i < arr.count; i ++) {
            int temp = [arr[i] integerValue];
            // 内层循环用于移动元素位置
            for (int j = i - 1; j >= 0 && temp < [arr[j] integerValue]; j --) {
                arr[j + 1] = arr[j];
                arr[j] = [NSNumber numberWithInt:temp];
            }
        }
        NSLog(@"插入排序结果：%@",arr);
    }

### 折半插入排序
折半插入排序利用了二分查找的优点：查找到元素的位置速度更快。
##### 算法思想
从第二个元素开始逐个置入监视哨，使用low、high标签进行折半判断比较大小，并确认插入位置，该位置到最后一个数全部后移一位，然后腾出该位置，把监视哨里面的数置入该位置。依此类推进行排序，直到最后一个数比较完毕。

    - (void)binaryInsertSort:(NSMutableArray *)arr{
        //索引从1开始 默认让出第一元素为默认有序表 从第二个元素开始比较
        for(int i = 1 ; i < [list count] ; i++){
            //binary search start
            NSInteger temp= [arr[i] intValue];
            int left = 0; 
            int right = i - 1;
            while (left <= right) {
                int middle = (left + right)/2;
                if(temp < [arr[middle] intValue]){   
                    right = middle - 1;    
                }else{     
                    left = middle + 1;
                }  
            }
            //binary search end
            for(int j = i ; j > left; j--){
                [arr replaceObjectAtIndex:j withObject:arr[j-1]]; 
            }
            [arr replaceObjectAtIndex:left withObject:[NSNumber numberWithInt:temp]];
        }  
    }

# 堆排序
> 堆是指二叉堆，二叉堆又称完全二叉树或者叫近似完全二叉树。二叉堆又分为最大堆和最小堆。
###### 最大堆的特性如下：
父结点的键值总是大于或者等于任何一个子节点的键值
每个结点的左子树和右子树都是一个最大堆
###### 最小堆的特性如下：
父结点的键值总是小于或者等于任何一个子节点的键值
每个结点的左子树和右子树都是一个最小堆

##### 最大堆的算法思想是：
先将初始的R[0…n-1]建立成最大堆，此时是无序堆，而堆顶是最大元素
再将堆顶R[0]和无序区的最后一个记录R[n-1]交换，由此得到新的无序区R[0…n-2]和有序区R[n-1]，且满足R[0…n-2].keys ≤ R[n-1].key
由于交换后，前R[0…n-2]可能不满足最大堆的性质，因此再调整前R[0…n-2]为最大堆，直到只有R[0]最后一个元素才调整完成。

最大堆排序完成后，其实是升序序列，每次调整堆都是要得到最大的一个元素，然后与当前堆的最后一个元素交换，因此最后所得到的序列是升序序列。
##### 最小堆的算法思想是：
先将初始的R[0…n-1]建立成最小堆，此时是无序堆，而堆顶元素是最小的元素
再将堆顶R[0]与无序区的最后一个R[n-1]交换，由此得到新的无序堆R[0…n-2]和有序堆R[n-1]，且满足R[0…n-2].keys >= R[n-1].key
由于交换后，前R[0…n-2]可能不满足最小堆的性质，因此再调整前R[0…n-2]为最小堆，直到只有R[0]最后一个元素才调整完成

最小堆排序完成后，其实是降序序列，每次调整堆都是要得到最小的一个元素，然后与当前无序堆的最后一个元素交换，所以所得到的序列是降序的。

    - (void)heapSortwithOrder:(BOOL)isAsc {
        for (NSInteger i = arr.count/2 - 1; i>=0; --i) {
            [self siftWithLow:i high:arr.count asc:isAsc];
        }
        for (NSInteger i = arr.count - 1; i>=1; --i) {
            id temp = arr[0];
            arr[0] = arr[i];
            arr[i] = temp;
            [self siftWithLow:0 high:i asc:isAsc];
         }
     }
    - (void)siftWithLow:(NSInteger)low high:(NSInteger)high asc:(BOOL)isAsc {
        NSInteger left = 2 * low + 1;
        NSInteger right = left + 1;
        NSInteger lastIndex = low;
        //左子节点大的情况 
        if (right < high && ((arr[right] > arr[lastIndex] && isAsc) || (arr[right] < arr[lastIndex] && !isAsc))) {
            lastIndex = left;
        }
        //右子节点大的情况
        if (right < high && ((arr[right] > arr[lastIndex] && isAsc) || (arr[right] < arr[lastIndex] && !isAsc))) {
            lastIndex = right;
        }
        //节点改变
        if (lastIndex != low) {
            //较大的节点值将交换到其所在节点的父节点
            id temp = arr[low];
            arr[low] = arr[lastIndex];
            arr[lastIndex] = temp;
            //递归遍历
            [self siftWithLow:lastIndex high:high asc:isAsc];
        }
    }

# 参考
> [排序算法(OC实现)](http://blog.csdn.net/alpaca12/article/details/52038644)
