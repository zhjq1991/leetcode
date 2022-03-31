#### 9、用两个栈实现队列

```C++
class CQueue {
private:
    stack<int> inStack, outStack;

    void in2out() {
        while (!inStack.empty()) {
            outStack.push(inStack.top());
            inStack.pop();
        }
    }

public:
    CQueue() {}

    void appendTail(int value) {
        inStack.push(value);
    }

    int deleteHead() {
        if (outStack.empty()) {
            if (inStack.empty()) {
                return -1;
            }
            in2out();
        }
        int value = outStack.top();
        outStack.pop();
        return value;
    }
};
```

#### 10、 