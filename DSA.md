## 🔢 Arrays

```
public class ArrayDemo {
    public static void main(String[] args) {
        int[] arr = {1, 2, 3, 4};
        for (int i = 0; i < arr.length; i++) {
            System.out.println(arr[i]);
        }
    }
}
```

Dynamic Array:

```
import java.util.ArrayList;

public class DynamicArrayDemo {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList<>();
        list.add(10);
        list.add(20);
        list.remove(0);
        for (int x : list) System.out.println(x);
    }
}
```

---

## 🔗 Linked Lists

Singly:

```
class Node {
    int data;
    Node next;
    Node(int d){ data=d; }
}

public class SinglyLinkedList {
    Node head;
    void insert(int d){
        Node n=new Node(d);
        if(head==null){ head=n; return; }
        Node cur=head;
        while(cur.next!=null) cur=cur.next;
        cur.next=n;
    }
    void print(){
        Node cur=head;
        while(cur!=null){ System.out.println(cur.data); cur=cur.next; }
    }
    public static void main(String[] args){
        SinglyLinkedList list=new SinglyLinkedList();
        list.insert(1); list.insert(2); list.insert(3);
        list.print();
    }
}
```

Doubly:

```
class DNode {
    int data;
    DNode prev, next;
    DNode(int d){ data=d; }
}

public class DoublyLinkedList {
    DNode head;
    void insert(int d){
        DNode n=new DNode(d);
        if(head==null){ head=n; return; }
        DNode cur=head;
        while(cur.next!=null) cur=cur.next;
        cur.next=n; n.prev=cur;
    }
    void printForward(){
        DNode cur=head;
        while(cur!=null){ System.out.println(cur.data); cur=cur.next; }
    }
    void printBackward(){
        DNode cur=head;
        while(cur.next!=null) cur=cur.next;
        while(cur!=null){ System.out.println(cur.data); cur=cur.prev; }
    }
    public static void main(String[] args){
        DoublyLinkedList list=new DoublyLinkedList();
        list.insert(1); list.insert(2); list.insert(3);
        list.printForward();
        list.printBackward();
    }
}
```

Circular:

```
class CNode {
    int data;
    CNode next;
    CNode(int d){ data=d; }
}

public class CircularLinkedList {
    CNode head;
    void insert(int d){
        CNode n=new CNode(d);
        if(head==null){ head=n; head.next=head; return; }
        CNode cur=head;
        while(cur.next!=head) cur=cur.next;
        cur.next=n; n.next=head;
    }
    void print(){
        if(head==null) return;
        CNode cur=head;
        do{ System.out.println(cur.data); cur=cur.next; } while(cur!=head);
    }
    public static void main(String[] args){
        CircularLinkedList list=new CircularLinkedList();
        list.insert(1); list.insert(2); list.insert(3);
        list.print();
    }
}
```

---

## 📚 Stacks

Array-based:

```
class StackArray {
    int[] arr=new int[5]; int top=-1;
    void push(int x){ arr[++top]=x; }
    int pop(){ return arr[top--]; }
    boolean isEmpty(){ return top==-1; }
    public static void main(String[] args){
        StackArray s=new StackArray();
        s.push(10); s.push(20);
        while(!s.isEmpty()) System.out.println(s.pop());
    }
}
```

Linked List:

```
class StackNode {
    int data; StackNode next;
    StackNode(int d){ data=d; }
}

class StackLL {
    StackNode top;
    void push(int x){ StackNode n=new StackNode(x); n.next=top; top=n; }
    int pop(){ int v=top.data; top=top.next; return v; }
    boolean isEmpty(){ return top==null; }
    public static void main(String[] args){
        StackLL s=new StackLL();
        s.push(10); s.push(20);
        while(!s.isEmpty()) System.out.println(s.pop());
    }
}
```

---

## 📦 Queues

Array-based:

```
class QueueArray {
    int[] arr=new int[5]; int front=0,rear=0;
    void enqueue(int x){ arr[rear++]=x; }
    int dequeue(){ return arr[front++]; }
    boolean isEmpty(){ return front==rear; }
    public static void main(String[] args){
        QueueArray q=new QueueArray();
        q.enqueue(1); q.enqueue(2);
        while(!q.isEmpty()) System.out.println(q.dequeue());
    }
}
```

Linked List:

```
class QNode { int data; QNode next; QNode(int d){data=d;} }

class QueueLL {
    QNode front,rear;
    void enqueue(int x){
        QNode n=new QNode(x);
        if(rear==null){ front=rear=n; return; }
        rear.next=n; rear=n;
    }
    int dequeue(){ int v=front.data; front=front.next; if(front==null) rear=null; return v; }
    boolean isEmpty(){ return front==null; }
    public static void main(String[] args){
        QueueLL q=new QueueLL();
        q.enqueue(1); q.enqueue(2);
        while(!q.isEmpty()) System.out.println(q.dequeue());
    }
}
```

Circular Queue:

```
class CircularQueue {
    int[] arr=new int[5]; int front=0,rear=0,size=0;
    void enqueue(int x){ if(size==arr.length) return; arr[rear]=(x); rear=(rear+1)%arr.length; size++; }
    int dequeue(){ int v=arr[front]; front=(front+1)%arr.length; size--; return v; }
    boolean isEmpty(){ return size==0; }
    public static void main(String[] args){
        CircularQueue q=new CircularQueue();
        q.enqueue(1); q.enqueue(2); q.enqueue(3);
        while(!q.isEmpty()) System.out.println(q.dequeue());
    }
}
```

Deque:

```
import java.util.ArrayDeque;
public class DequeDemo {
    public static void main(String[] args){
        ArrayDeque<Integer> dq=new ArrayDeque<>();
        dq.addFirst(1); dq.addLast(2);
        System.out.println(dq.removeFirst());
        System.out.println(dq.removeLast());
    }
}
```

Priority Queue:

```
import java.util.PriorityQueue;
public class PriorityQueueDemo {
    public static void main(String[] args){
        PriorityQueue<Integer> pq=new PriorityQueue<>();
        pq.add(30); pq.add(10); pq.add(20);
        while(!pq.isEmpty()) System.out.println(pq.poll());
    }
}
```

---

## 🌳 Trees

Binary Tree:

```
class TreeNode { int data; TreeNode left,right; TreeNode(int d){data=d;} }

public class BinaryTree {
    TreeNode root;
    void inorder(TreeNode n){ if(n!=null){ inorder(n.left); System.out.println(n.data); inorder(n.right);} }
    public static void main(String[] args){
        BinaryTree t=new BinaryTree();
        t.root=new TreeNode(1);
        t.root.left=new TreeNode(2);
        t.root.right=new TreeNode(3);
        t.inorder(t.root);
    }
}
```

Binary Search Tree:

```
class BST {
    TreeNode root;
    TreeNode insert(TreeNode n,int x){
        if(n==null) return new TreeNode(x);
        if(x<n.data) n.left=insert(n.left,x);
        else n.right=insert(n.right,x);
        return n;
    }
    void inorder(TreeNode n){ if(n!=null){ inorder(n.left); System.out.println(n.data); inorder(n.right);} }
    public static void main(String[] args){
        BST bst=new BST();
        bst.root=bst.insert(bst.root,5);
        bst.insert(bst.root,3); bst.insert(bst.root,7);
        bst.inorder(bst.root);
    }
}
```

AVL, Red-Black, Segment Tree, Fenwick Tree are more advanced — I can expand those with full balancing logic if you want, but they’re quite long.

---

## 📖 Trie

```
class TrieNode { TrieNode[] children=new TrieNode[26]; boolean end; }

class Trie {
    TrieNode root=new TrieNode();
    void insert(String word){
        TrieNode node=root;
        for(char c:word.toCharArray()){
            int i=c-'a';
            if(node.children[i]==null) node.children[i]=new TrieNode();
            node=node.children[i];
        }
        node.end=true;
    }
    boolean search(String word){
        TrieNode node=root;
        for(char c:word.toCharArray()){
            int i=c-'a';
            if(node.children[i]==null) return false;
            node
```

---

## 🌐 Graphs

### 1. **Adjacency List (Undirected)**

```
import java.util.*;

public class GraphAdjList {
    private Map<Integer, List<Integer>> adj = new HashMap<>();

    void addEdge(int u, int v) {
        adj.putIfAbsent(u, new ArrayList<>());
        adj.putIfAbsent(v, new ArrayList<>());
        adj.get(u).add(v);
        adj.get(v).add(u); // undirected
    }

    void printGraph() {
        for (var entry : adj.entrySet()) {
            System.out.println(entry.getKey() + " -> " + entry.getValue());
        }
    }

    public static void main(String[] args) {
        GraphAdjList g = new GraphAdjList();
        g.addEdge(1, 2);
        g.addEdge(1, 3);
        g.addEdge(2, 4);
        g.printGraph();
    }
}
```

---

### 2. **Adjacency Matrix (Directed)**

```
public class GraphAdjMatrix {
    int[][] matrix;
    int V;

    GraphAdjMatrix(int V) {
        this.V = V;
        matrix = new int[V][V];
    }

    void addEdge(int u, int v) {
        matrix[u][v] = 1; // directed
    }

    void printGraph() {
        for (int i = 0; i < V; i++) {
            for (int j = 0; j < V; j++) {
                System.out.print(matrix[i][j] + " ");
            }
            System.out.println();
        }
    }

    public static void main(String[] args) {
        GraphAdjMatrix g = new GraphAdjMatrix(4);
        g.addEdge(0, 1);
        g.addEdge(0, 2);
        g.addEdge(1, 3);
        g.printGraph();
    }
}
```

---

### 3. **Weighted Graph (Adjacency List)**

```
import java.util.*;

class Edge {
    int to, weight;
    Edge(int t, int w) { to = t; weight = w; }
}

public class WeightedGraph {
    private Map<Integer, List<Edge>> adj = new HashMap<>();

    void addEdge(int u, int v, int w) {
        adj.putIfAbsent(u, new ArrayList<>());
        adj.get(u).add(new Edge(v, w));
    }

    void printGraph() {
        for (var entry : adj.entrySet()) {
            System.out.print(entry.getKey() + " -> ");
            for (Edge e : entry.getValue()) {
                System.out.print("(" + e.to + ", " + e.weight + ") ");
            }
            System.out.println();
        }
    }

    public static void main(String[] args) {
        WeightedGraph g = new WeightedGraph();
        g.addEdge(1, 2, 5);
        g.addEdge(1, 3, 10);
        g.addEdge(2, 4, 2);
        g.printGraph();
    }
}
```

---

### 4. **Directed Graph**

```
import java.util.*;

public class DirectedGraph {
    private Map<Integer, List<Integer>> adj = new HashMap<>();

    void addEdge(int u, int v) {
        adj.putIfAbsent(u, new ArrayList<>());
        adj.get(u).add(v); // directed only
    }

    void printGraph() {
        for (var entry : adj.entrySet()) {
            System.out.println(entry.getKey() + " -> " + entry.getValue());
        }
    }

    public static void main(String[] args) {
        DirectedGraph g = new DirectedGraph();
        g.addEdge(1, 2);
        g.addEdge(1, 3);
        g.addEdge(2, 4);
        g.printGraph();
    }
}
```

---

### 5. **Undirected Graph**

```
import java.util.*;

public class UndirectedGraph {
    private Map<Integer, List<Integer>> adj = new HashMap<>();

    void addEdge(int u, int v) {
        adj.putIfAbsent(u, new ArrayList<>());
        adj.putIfAbsent(v, new ArrayList<>());
        adj.get(u).add(v);
        adj.get(v).add(u); // undirected
    }

    void printGraph() {
        for (var entry : adj.entrySet()) {
            System.out.println(entry.getKey() + " -> " + entry.getValue());
        }
    }

    public static void main(String[] args) {
        UndirectedGraph g = new UndirectedGraph();
        g.addEdge(1, 2);
        g.addEdge(2, 3);
        g.addEdge(3, 4);
        g.printGraph();
    }
}
```

