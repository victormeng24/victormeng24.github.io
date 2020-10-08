---
title: LeetCode--LRU Cache
date: 2019-06-20 12:32:27
tags:
- 算法
- 数据结构
---
# Description

Design and implement a data structure for Least Recently Used (LRU) cache. It should support the following operations: get and set.

get(key) - Get the value (will always be positive) of the key if the key exists in the cache, otherwise return -1.

set(key, value) - Set or insert the value if the key is not already present. When the cache reached its capacity, it should invalidate the least recently used item before inserting a new item。
# Solution

一道经典面试题
实现一个简单的LRU队列，对外暴露存取两种操作即可；

需要注意的是，队列一定是有capacity的，所以当队列到达容量上限时，需要根据LRU(Least Recently Used)规则淘汰掉最近最少使用的key；

解决思路：双链表+hashmap

双链表中的节点亦存储于HashMap中，保证O(1)的复杂度；
双链表支持删除结点操作和将结点插入到队首两个操作；
调用get&&set时，节点直接入队首，表示最近使用；
Capacity到达阈值时淘汰尾节点&&删除HashMap对应的Key；

参考：https://discuss.leetcode.com/topic/34701/java-easy-version-to-understand/2

# Code
最近搬运自己之前的blog，发现code都没format。。后续补一下
```js
public class LRUCache {
    int capacity;
    Map<Integer,Node> map;
    Node head;
    Node tail;
    int nums;
    class Node{
        public Node(int key,int val){
            this.key=key;
            this.value=val;
        }
        int key;
        int value;
        Node pre;
        Node next;
    }
    public LRUCache(int capacity) {
        this.capacity=capacity;
        map=new HashMap<>();
        head=new Node(-1,-1);
        tail=new Node(-1,-1);
        head.next=tail;
        tail.pre=head;
        this.nums=0;
    }
    public void DeleteNode(Node node){
        node.pre.next=node.next;
        node.next.pre=node.pre;
    }
    public void addToHead(Node node){
        node.next=head.next;
        head.next.pre=node;
        head.next=node;
        node.pre=head;
    }
    public int get(int key) {
        Node node=map.get(key);
        if(node!=null){
           DeleteNode(node);
            addToHead(node);
            return node.value;
        }
        else return -1;
    }

    public void set(int key, int value) {
         if(map.get(key)==null){
             Node node = new Node(key, value);
             map.put(key,node);
             if(nums<capacity) {
                 nums++;
                 addToHead(node);
             }
             else{
                 map.remove(tail.pre.key);
                 DeleteNode(tail.pre);
                 addToHead(node);
             }
         }
        else{
             Node node=map.get(key);
             node.value=value;
             DeleteNode(node);
             addToHead(node);
         }
    }
}
```
