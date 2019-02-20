https://github.com/LRH1993/android_interview/blob/master/android/basis/message-mechanism.md

问题：

1.运行以下代码，查看结果

```java
public class Main2Activity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main2);
        new SubThread("SubThread-1").start();
    }

    class SubThread extends Thread {
        private static final String TAG = "SubThread";
        public Handler handler = new Handler(); // ① 处

        public SubThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            Handler handler = new Handler(); // ② 处
        }
    }

}
```

A.①运行时错误

B.②运行时错误

C.正常运行

2.运行以下代码，查看输出结果

```java
public class Main2Activity extends AppCompatActivity {
    private static final String TAG = "Main2Activity";

    Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            if (msg.what == 0x2) {
                Log.i(TAG, "① - " + msg.obj);// 这里打印消息内容
            }
            super.handleMessage(msg);
        }
    };
    SubThread subThread = new SubThread("SubThread-1");

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main2);
        new SubThreadNew("SubThreadNew-1").start();

    }

    class SubThread extends Thread {
        private static final String TAG = "SubThread";
        public Handler handler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                if (msg.what == 0x2) {
                    Log.i(TAG, "② - " + msg.obj);// 这里打印消息内容
                }
                super.handleMessage(msg);
            }
        };

        public SubThread(String name) {
            super(name);
        }

        @Override
        public void run() {
        }
    }

    class SubThreadNew extends Thread {
        private static final String TAG = "SubThread";

        public SubThreadNew(String name) {
            super(name);
        }

        @Override
        public void run() {
            Message obtain = Message.obtain(subThread.handler);
            obtain.what = 0x2;
            obtain.obj = "③";
            handler.sendMessage(obtain);// 这里发送内容为”③“的消息

            Message obtain2 = Message.obtain(handler);
            obtain2.what = 0x2;
            obtain2.obj = "④";
            subThread.handler.sendMessage(obtain2);// 这里发送内容为”④“的消息
        }
    }

}
```

A. Main2Activity: ① - ③

​	SubThread: ② - ④

B. Main2Activity: ① - ④

​	SubThread: ② - ③

C.运行错误







------

答案：

1.B

考察：要想使用消息机制，首先要创建一个Looper。

分析：②处是在子线程运行，未创建Looper，会抛”Can't create handler inside thread that has not called Looper.prepare()“；①处在主线程运行，Android启动时主线程(**ActivityThread**)已经创建了一个Looper。

```java
public static void main(String[] args) {
..........................
        Looper.prepareMainLooper();
  ..........................
        Looper.loop();
  ..........................

    }
```

2.A

考察：Message持有的Handler

分析：虽然 Message.obtain(subThread.handler) 和 Message.obtain(handler)指定了Message的handler，但是在加到队列之前会被置为调用 sendMessage 方法的handler 

```java
public class Handler {
    ........
    // 所有发送消息的方法最中都会调用到这里，向Looper中的MessageQueue中插入message
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;// 就是这里重置 message的handler的
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
    ........
}
```