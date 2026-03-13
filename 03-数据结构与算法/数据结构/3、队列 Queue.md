

# 2、代码实现

```java
public class MyQueuePlus {
    public static void main(String[] args) {
        //创建一个队列
        QueuePlus queue = new QueuePlus(4);// 此处有效数据为3
        char key = ' '; //接收用户输入
        Scanner scanner = new Scanner(System.in);//
        boolean loop = true;
        //输出一个菜单
        while (loop) {
            System.out.println("s(show): 显示队列");
            System.out.println("e(exit): 退出程序");
            System.out.println("a(add): 添加数据到队列");
            System.out.println("g(get): 从队列取出数据");
            System.out.println("h(head): 查看队列头的数据");
            key = scanner.next().charAt(0);//接收一个字符
            switch (key) {
                case 's':
                    queue.list();
                    break;
                case 'a':
                    System.out.println("输出一个数");
                    int value = scanner.nextInt();
                    queue.add(value);
                    break;
                case 'g': //取出数据
                    try {
                        int res = queue.get();
                        System.out.printf("取出的数据是%d\n", res);
                    } catch (Exception e) {
                        // TODO: handle exception
                        System.out.println(e.getMessage());
                    }
                    break;
                case 'e': //退出
                    scanner.close();
                    loop = false;
                    break;
                default:
                    break;
            }
        }

        System.out.println("程序退出~~");
    }

}

class QueuePlus{
    int rear;
    int front;
    int [] arr;
    int maxSize;

    public QueuePlus(int maxSize){
        this.maxSize = maxSize;
        arr = new int[maxSize];
    }


    public void add(int num){
        if (isFull()){
            throw new RuntimeException("队列满");
        }
        arr[rear] = num;
        rear = (rear + 1) % maxSize;
    }

    public int get(){
        if (isEmpty()){
            throw new RuntimeException("队列空");
        }
        int num = arr[front];
        front = (front + 1) % maxSize;
        return num;
    }

    public boolean isEmpty(){
        return front == rear;
    }

    public boolean isFull(){
        return (rear + 1) % maxSize == front;
    }

    public int size(){
        return (rear - front + maxSize) % maxSize;
    }

    public void list(){
        for (int i = front; i < front + size(); i++) {
            System.out.println(arr[i % maxSize]);
        }
    }
}
```

