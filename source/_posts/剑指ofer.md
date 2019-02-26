---
title: 剑指ofer
abbrlink: 53933
date: 2019-01-09 22:16:02
tags:
---

# T： 时间复杂度                  S：空间复杂度

## n个数，大小0-(n-1)，判断是否有重复

条件: 无

方法一: 排序，扫描即可        O(nlogn)

方法二: 使用HashTable         O(n)

方法三: 扫描下标为i的数字m时，判断m== i, 是，继续扫描，否，将m和下标为m的数字n进行比较,若相等，则重复，否则交换下标为m和下标为i的两个数字。

T: O(n)

S: O(1)

```
bool duplicate(int numbers[], int length, int *duplication) {
    if(numbers == nullptr || length <=0) {
        return false;
    }

//    判断输入是否符合条件
    for(int i = 0; i< length; i++) {
        if(numbers[i] < 0 || numbers[i] > length - 1) {
            return false;
        }
    }

//    正式处理
    int temp = 0;
    for(int i = 0; i < length; ++i) {
        while(numbers[i] != i) {
//            有重复数字
            if(numbers[i] == numbers[numbers[i]]) {
                *duplication = numbers[i];
                return true;
            }
//            交换标为i 和下标为numbers[i]的两个数字
            temp = numbers[i];
            numbers[i] = numbers[numbers[i]];
            numbers[numbers[i]] = temp;
        }
    }

    return false;
}
```

## n+1个数，大小1-n，判断是否有重复

条件: 不修改原数组

方法一: 开辟一个新的数组，在使用上边的方法            T: O(n)               S:O(n)

方法二: 把1-n的数字，从中间的数字m分为两部分，前一半为1-m,后一半为(m+1)-n,如果1~m的数字的数目超过m,那这一半区间存在重复。类似于二分法。

```
bool duplicate(int numbers[], int length, int *duplication) {
    if(numbers == nullptr || length <=0) {
        return false;
    }

//    判断输入是否符合条件
    for(int i = 0; i< length; i++) {
        if(numbers[i] < 1 || numbers[i] > length - 1) {
            return false;
        }
    }
// 4
// 1 1 2 3
//    正式处理
    int start = 1;
    int end = length - 1;
    int middle = 0;
    int count = 0;
    while(end >= start) {
        middle = ((end - start) >> 1) + start;
        count = countRange(numbers, length, start, middle);
        if(end == start) {
            if(count > 1) {
                *duplication = start;
                return true;
            } else {
                break;
            }
        }
//        计算差值
        if(count > (middle - start + 1)) {
            end = middle;
        } else {
            start = middle + 1;
        }
    }

    *duplication = start;
    return false;
}

int countRange(int numbers[], int length, int start, int end) {
    int count = 0;
    for(int i = 0; i< length; i++) {
        if(numbers[i] >= start && numbers[i] <=end) {
            count++;
        }
    }

    return count;
}
```

## 将空格替换为%20

##时间复杂度

T: O(n) 首先遍历一遍字符串，找出所有的空格数，统计出最终字符串长度。

```
#include <stdio.h>
#include <assert.h>

void ReplaceBlank(char str[], int length, int limit);

int main() {
	char str[50];
	printf("请输入一个字符串:");
	gets(str);
	printf("字符串: %s\n", str);
	int length = 0;
	length = strlen(str);
	printf("长度为:%d", length);
	ReplaceBlank(str, length, 50);
	printf("转换后的字符串:%s", str);
	return 0;
}

void ReplaceBlank(char str[], int length, int limit) {
	if(length == 0) {
		return;
	}
	int blanknumber = 0;
	for(int i = 0; i< length; i++) {
		if(str[i] == 32) {
			blanknumber++;
		}
	}

	int replaceStrLength = length + blanknumber * 2;
	if(replaceStrLength >= 50) {
		printf("超出范围");
		return;
	}
	str[replaceStrLength] = '\0';
	for(int i = length - 1; i >= 0; i--) {
		if(str[i] == 32) {
			str[replaceStrLength - 3] = '%';
			str[replaceStrLength - 2] = '2';
			str[replaceStrLength - 1] = '0';
			replaceStrLength -= 3;
		} else {
			str[replaceStrLength - 1] = str[i];
			replaceStrLength -= 1;
		}
	}
}
```

链表:

```
//从尾到头打印链表
//使用递归，容易出现堆栈溢出
//使用栈
void PrintListReversingly_Iteratively(ListNode *pHead) {
    int result[20] = {0};
    ListNode *pNode = pHead;
    int length = 0;
    while(pNode != NULL) {
        result[length] = pNode->m_nValue;
        length++;
        pNode = pNode->m_pNext;
    }

    for(int i = length - 1; i >= 0; i--) {
        printf("%d\t", result[i]);
    }
}
//
//void Recursively(ListNode *pHead) {
//    if(pHead != NULL) {
//        if(pHead->m_pNext != NULL) {
//            Recursively(pHead->m_pNext);
//        }
//
//        printf("%d\t", pHead->m_nValue);
//    }
//}
```

