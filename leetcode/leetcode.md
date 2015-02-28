#LeetCode Notes
------

早就说要练习LeetCode了，拿来练练大脑

------
[TOC]

##Remove Elements
Given an array and a value, remove all instances of that value in place and return the new length.
The order of elements can be changed. It doesn't matter what you leave beyond the new length.
在原地删除指定元素，用一个num记录长度，遍历数组不相等的就替换num与i下标，相等的就被in place替换。
```cpp
int removeElem1(int *a,int n,int e)
{
    int i =0;
    int num=0;
    for(int i=0;i<n;i++)
    {
        if(a[i] != e)
        {
            a[num]=a[i];
            num++;
        }
    }
    return num;
}
```
另外，使用stl的remove算法也可以方便remove容器内的指定元素
```cpp
    int a[]={1,2,3,4,5,6,7,5,6,7,5};
    vector<int> va(&a[0],&a[11]);
    copy(va.begin(),va.end(),ostream_iterator<int>(cout,","));
    //需要注意的是，remove往往和erase搭档，前者并不删除元素
    va.erase(remove(va.begin(),va.end(),5),va.end());
    cout <<endl;
    copy(va.begin(),va.end(),ostream_iterator<int>(cout,","));
```

##Remove Duplicates from Sorted Array
Given a sorted array, remove the duplicates in place such that each element appear only once and return the new length. Do not allocate extra space for another array, you must do this in place with constant memory.
```cpp
int removeDuplicate(int*a,int n)
{
    int num = 0;
    for(int i=1;i<n;i++)
    {
        if(a[i-1]!=a[i])
        {
            a[num++]=a[i-1];
        }
    }
    return num;
}
```
考虑利用num作为新的数组，我们只要将不重复的元素放到num为index的位置即可