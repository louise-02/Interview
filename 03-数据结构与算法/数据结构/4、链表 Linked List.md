# 1、相关概念

链表是以节点的方式进行存储的。

每个节点包括 Data 域和 Next 域指向下一个节点。

链表的各个节点不一定是连续存储。

链表分带头节点的链表和没有头节点的链表。

# 2、单链表增删改

```java
class LinkedList {

    Person headPerson;

    public LinkedList() {
        this.headPerson = new Person();
    }

    public void add(Person addPerson) {
        Person person = headPerson;
        Long id = addPerson.getId();
        while (true) {
            if (person.nextPerson != null && person.nextPerson.getId() >= id) {
                addPerson.nextPerson = person.nextPerson;
                person.nextPerson = addPerson;
                break;
            }
            if (person.nextPerson == null) {
                person.nextPerson = addPerson;
                break;
            }
            person = person.nextPerson;
        }
    }

    public void update(Person updatePerson) {
        Person temp = headPerson;
        Long id = updatePerson.getId();
        while (true) {
            if (temp.nextPerson == null) {
                break;
            }
            if (Objects.equals(temp.getId(), id)) {
                temp.setName(updatePerson.getName());
                break;
            }
            temp = temp.nextPerson;
        }
    }

    public void delete(Long id) {
        Person temp = headPerson;

        while (true) {
            if (temp.nextPerson == null) {
                break;
            }
            if (Objects.equals(temp.nextPerson.id, id)) {
                temp.nextPerson = temp.nextPerson.nextPerson;
                break;
            }
            temp = temp.nextPerson;
        }
    }


    public void list() {
        Person person = headPerson.nextPerson;
        while (true) {
            System.out.println(person);
            if (person.nextPerson == null) {
                break;
            }
            person = person.nextPerson;
        }
    }

}
```

# 3、单链表一些面试题

## 获取链表大小

```java
public int size(){
    Person temp = headPerson;
    int size = 0;
    if (temp.nextPerson == null){
        return size;
    }
    temp = temp.nextPerson;

    while (temp != null){
        size++;
        temp = temp.nextPerson;
    }
    return size;
}
```

## 获取倒数第K位元素

```java
public Person getLastPerson(int lastNum){
    if (headPerson.nextPerson == null){
        return null;
    }
    int size = size();
    if (lastNum <= 0 || lastNum > size){
        return null;
    }
    Person temp = headPerson.nextPerson;
    for (int i = 0; i < size - lastNum; i++) {
        temp = temp.nextPerson;
    }
    return temp;
}
```

## 单链表反转

```java
public void reversal(){
    if (headPerson.nextPerson == null
            || headPerson.nextPerson.nextPerson == null){
        return;
    }

    Person nextPerson = headPerson.nextPerson;
    Person temp;
    Person newHead = new Person();
    while (nextPerson != null){
        temp = nextPerson.nextPerson;
        nextPerson.nextPerson = newHead.nextPerson;
        newHead.nextPerson = nextPerson;
        nextPerson = temp;
    }
    headPerson = newHead;
}
```

## 倒序输出

```java
public void reversalPrint(){
    Person nextPerson = headPerson.nextPerson;
    if (nextPerson == null){
        return;
    }
    Stack<Person> stack = new Stack<>();
    while (nextPerson != null){
        stack.add(nextPerson);
        nextPerson = nextPerson.nextPerson;
    }
    while (!stack.isEmpty()){
        System.out.println(stack.pop());
    }
}
```

## 合并两个有序链表

```java
public void addAll(LinkedList linkedList){
    Person otherHead = linkedList.headPerson;
    if (otherHead.nextPerson == null)
        return;
    Person thisPerson = this.headPerson.nextPerson;
    Person otherPerson = linkedList.headPerson.nextPerson;
    Person newHead = new Person();
    Person temp = newHead;
    Person thisTemp;
    Person otherTemp;
    while (true){
        if (thisPerson != null && otherPerson != null){
            if (thisPerson.getId() <= otherPerson.getId()){
                thisTemp = thisPerson.nextPerson;
                temp.nextPerson = thisPerson;
                thisPerson.nextPerson = null;
                thisPerson = thisTemp;
            }else {
                otherTemp = otherPerson.nextPerson;
                temp.nextPerson = otherPerson;
                otherPerson.nextPerson = null;
                otherPerson = otherTemp;
            }
            temp = temp.nextPerson;
            continue;
        }
        if (thisPerson != null){
            thisTemp = thisPerson.nextPerson;
            temp.nextPerson = thisPerson;
            thisPerson.nextPerson = null;
            thisPerson = thisTemp;
        }
        if (otherPerson != null){
            otherTemp = otherPerson.nextPerson;
            temp.nextPerson = otherPerson;
            otherPerson.nextPerson = null;
            otherPerson = otherTemp;
        }
        temp = temp.nextPerson;
        if (thisPerson == null && otherPerson == null){
            headPerson = newHead;
            break;
        }
    }
}
```

# 4、双向链表

**单链表的缺陷**

查询只有一个方向

无法自我删除

**实现**

定义一个前一个后其他相似

# 5、约瑟夫问题

```java
static class JosePhuNode<T>{
    Integer id;
    T data;
    JosePhuNode<T> nextNode;

    public JosePhuNode(Integer id, T data) {
        this.id = id;
        this.data = data;
    }

    @Override
    public String toString() {
        return "JosePhuNode{" +
                "id=" + id +
                ", data=" + data +
                '}';
    }
}


public void addNode(JosePhuNode<T> addNode){
    if (firstNode == null){
        firstNode = addNode;
        firstNode.nextNode = addNode;
        return;
    }

    JosePhuNode<T> temp = firstNode;
    while (true){
        if (temp.nextNode == firstNode){
            temp.nextNode = addNode;
            addNode.nextNode = firstNode;
            break;
        }
        temp = temp.nextNode;
    }
}

public void list(){
    if (firstNode == null){
        System.out.println("空链表");
        return;
    }

    JosePhuNode<T> temp = this.firstNode;
    do {
        System.out.println(temp);
        temp = temp.nextNode;
    } while (temp != firstNode);
}

//从start开始 每each个出
public void doJosePhu(JosePhu josePhu,int start,int each){
    if (josePhu.firstNode == null){
        return;
    }
    JosePhuNode lastNode = getLastNode(josePhu);

    for (int i = 1; i < start; i++) {
        josePhu.firstNode = josePhu.firstNode.nextNode;
        lastNode = lastNode.nextNode;
    }
    int num = 0;
    while (true){
        num++;
        if (num == each){
            lastNode.nextNode = josePhu.firstNode.nextNode;
            System.out.println(josePhu.firstNode);
            josePhu.firstNode = josePhu.firstNode.nextNode;
            continue;
        }
        num = num % each;
        josePhu.firstNode = josePhu.firstNode.nextNode;
        lastNode = lastNode.nextNode;
        if (josePhu.firstNode == lastNode){
            System.out.println("last: "+ lastNode);
            break;
        }
    }
}
```

