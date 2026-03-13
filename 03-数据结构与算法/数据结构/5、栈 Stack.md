# 1、相关概念

栈是一个先进后出的有序队列。

栈是限制线性表中元素的插入和删除，只能在线性表的同一段进行的一种特殊线性表，允许插入和删除的一端成为栈顶，另一固定的一端成为栈底。

# 2、应用场景

1、子程序的调用：在跳往子程序前，会先将下一个指令的地址存在栈中，直到子程序执行完后再讲地址取出，以回到原来的程序中。

2、处理递归调用：和子程序的调用类似，只是除了存储下一个指令的地址外，也将参数、区域变量等数据存入栈中。

3、表达式的转换[中缀表达式转后缀表达式]与求值。

4、二叉树的遍历。

5、图形的深度优化搜索法。

# 3、代码实现

```java
public class MyStack<T> {
    Object[] data;
    int maxSize;
    int point;

    public MyStack(int size) {
        this.maxSize = size;
        this.data = new Object[size];
        point = 0;
    }

    public MyStack() {
        this.data = new Object[10];
        this.maxSize = 10;
        point = 0;
    }

    public boolean isFull(){
        return point == data.length;
    }

    public int size(){
        return point;
    }

    public boolean isEmpty(){
        return point == 0;
    }
    
    //扩容
    public void dilatation(){
        this.data = Arrays.copyOf(data,maxSize * 2);
    }

    public void push(T entity){
        if (isFull()){
            System.err.println("is full");
            dilatation();
            System.err.println("do dilatation");
        }
        data[point] = entity;
        point ++;
    }

    @SuppressWarnings("unchecked")
    public T pop(){
        if (isEmpty()){
            return null;
        }
        Object entity = data[point - 1];
        data[point] = null;
        point--;
        return (T) entity;
    }


    public void list(){
        if (isEmpty()){
            return;
        }
        for (int i = 0; i < point; i++) {
            System.out.println(data[i]);
        }
    }

}
```

# 4、前缀表达式

前缀表达式又称波兰式，前缀表达式的运算符位于操作数之前。

例如：- × + 3 4 5 6

**前缀表达式求值**

从右至左扫描表达式，遇到数字时，将数字压入堆栈，遇到运算符时，弹出栈顶的两个数，用运算符对它们做相应的计算（栈顶元素 op 次顶元素），并将结果入栈；重复上述过程直到表达式最左端，最后运算得出的值即为表达式的结果。

## 中缀表达式转前缀表达式

```java
初始化两个栈:运算符栈s1，储存中间结果的栈s2
从右至左扫描中缀表达式
遇到操作数时，将其压入s2
遇到运算符时，比较其与s1栈顶运算符的优先级
    如果s1为空，或栈顶运算符为右括号“)”，则直接将此运算符入栈
    否则，若优先级比栈顶运算符的较高或相等，也将运算符压入s1
    否则，将s1栈顶的运算符弹出并压入到s2中，再次转到(4-1)与s1中新的栈顶运算符相比较
遇到括号时
    如果是右括号“)”，则直接压入s1
    如果是左括号“(”，则依次弹出S1栈顶的运算符，并压入S2，直到遇到右括号为止，此时将这一对括号丢弃
重复步骤2至5，直到表达式的最左边
将s1中剩余的运算符依次弹出并压入s2
依次弹出s2中的元素并输出，结果即为中缀表达式对应的前缀表达式

1+((2+3)×4)-5     - + 1 × + 2 3 4 5
```

代码实现

```java
@SuppressWarnings("all")
public class PrefixExpression {
    private MyStack<String> symbolStack = new MyStack<>();
    private MyStack<String> numStack = new MyStack<>();

    private static final String add = "+";
    private static final String minus = "-";
    private static final String mult = "×";
    private static final String division = "/";
    private static final String leftBracket = "(";
    private static final String rightBracket = ")";

    private static List<String> operatorList = new ArrayList<>();

    static {
        operatorList.add(add);
        operatorList.add(minus);
        operatorList.add(mult);
        operatorList.add(division);
    }

    @Test
    public void convertToPrefix() {
        //String expression = "1+((2+3)×4)-5";
        //String expression = "1-2+(3+2-7)×5+6";
        String expression = "5/6+(3×6×(3+2)-5)-1";
        char[] chars = expression.toCharArray();
        for (int i = chars.length - 1; i >= 0; i--) {
            doResolve(new Character(chars[i]).toString());
        }
        while (StrUtil.isNotEmpty(symbolStack.getOne())) {
            numStack.push(symbolStack.pop());
        }
        int num = calculate(numStack);
        System.err.println(num);
    }

    private int calculate(MyStack<String> numStack) {
        numStack = numStack.reversal();
        while (true) {
            if (numStack.isEmpty()){
                break;
            }
            String one = numStack.pop();
            if (isNum(one)) {
                symbolStack.push(one);
                continue;
            }
            if (isOperator(one)) {
                String pop1 = symbolStack.pop();
                String pop2 = symbolStack.pop();
                String str = doCalculate(pop1, pop2, one);
                symbolStack.push(str);
            }
        }
        return Integer.valueOf(symbolStack.pop());
    }

    private String doCalculate(String pop1, String pop2, String symbol) {
        switch (symbol){
            case add:
                return ((Integer)(Integer.valueOf(pop1) + Integer.valueOf(pop2))).toString();
            case minus:
                return ((Integer)(Integer.valueOf(pop1) - Integer.valueOf(pop2))).toString();
            case division:
                return ((Integer)(Integer.valueOf(pop1) / Integer.valueOf(pop2))).toString();
            case mult:
                return ((Integer)(Integer.valueOf(pop1) * Integer.valueOf(pop2))).toString();
        }
        return null;
    }


    public void doResolve(String str) {
        if (isNum(str)) {
            numStack.push(str);
            return;
        }
        if (isOperator(str)) {
            pushSymbol(str);
            return;
        }
        if (rightBracket.equals(str)) {
            symbolStack.push(str);
            return;
        }
        if (leftBracket.toString().equals(str)) {
            while (!rightBracket.toString().equals(symbolStack.getOne())) {
                numStack.push(symbolStack.pop());
            }
            symbolStack.pop();
            return;
        }
    }

    private void pushSymbol(String str) {
        int value1 = getSymbolValue(str);

        if (StrUtil.isEmpty(symbolStack.getOne())) {
            symbolStack.push(str);
            return;
        }

        while (value1 < getSymbolValue(symbolStack.getOne())) {
            numStack.push(symbolStack.pop());
        }
        symbolStack.push(str);
    }

    private int getSymbolValue(String str) {
        switch (str) {
            case add:
                return 1;
            case minus:
                return 1;
            case mult:
                return 2;
            case division:
                return 2;
            case rightBracket:
                return 0;
            case leftBracket:
                return 0;
            default:
                throw new RuntimeException("123");
        }
    }

    public boolean isNum(String str) {
        strIsEmpty(str);
        return str.matches("[\\d]+");
    }

    public boolean isOperator(String str) {
        strIsEmpty(str);
        return operatorList.contains(str);
    }


    private void strIsEmpty(String str) {
        if (str == null || str.length() < 0)
            throw new NullPointerException();
    }
```

# 5、后缀表达式

与前缀表达式类似，只是顺序是从左至右。

3 4 + 5 × 6 -

**后缀表达式求值**

从左至右扫描表达式，遇到数字时，将数字压入堆栈，遇到运算符时，弹出栈顶的两个数，用运算符对它们做相应的计算（次顶元素 op 栈顶元素），并将结果入栈；重复上述过程直到表达式最右端，最后运算得出的值即为表达式的结果。

## 中缀表达式转后缀表达式

```java
初始化两个栈：运算符栈s1和储存中间结果的栈s2；
从左至右扫描中缀表达式；
遇到操作数时，将其压s2；
遇到运算符时，比较其与s1栈顶运算符的优先级
    如果s1为空，或栈顶运算符为左括号“(”，则直接将此运算符入栈；
    否则，若优先级比栈顶运算符的高，也将运算符压入s1（注意转换为前缀表达式时是优先级较高或相同，而这里则不包括相同的情况）；
    否则，将s1栈顶的运算符弹出并压入到s2中，再次转到(4-1)与s1中新的栈顶运算符相比较；
遇到括号时：
    如果是左括号“(”，则直接压入s1；
    如果是右括号“)”，则依次弹出s1栈顶的运算符，并压入s2，直到遇到左括号为止，此时将这一对括号丢弃；
重复步骤2至5，直到表达式的最右边；
将s1中剩余的运算符依次弹出并压入s2；
依次弹出s2中的元素并输出，结果的逆序即为中缀表达式对应的后缀表达式（转换为前缀表达式时不用逆序）

1+((2+3)×4)-5       1 2 3 + 4 × + 5 -
```

