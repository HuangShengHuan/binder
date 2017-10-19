# Binder驱动

Binder驱动是由系统创建的C++进程，专门用来接收和转发其他进程的通讯信息；

## 原理

Binder驱动会为每一个进程，根据其pid分配一份内存，而每个进程会把自己的aidl一一注册进这份内存中；

在客户端进程需要访问服务端进程时，会将需要访问的服务端的aidl信息传进Binder驱动中（应该是通过**bindService\(intent\)**），Binder驱动根据客户端的访问信息，在内存中查找对应aidl的服务端进程，然后将信息传递给服务端进程。

服务端进程在接收信息之后，会创建一个Binder，并通过Binder驱动传递回来，最终交给了客户端进程。

1. 客户端向Binder驱动发出请求，与Binder驱动建立链接：

```
Intent intent = new Intent();
intent.setPackage("com.example.bindertest.aidl.IBInderInterface");
bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
```

    2. Binder驱动中找到存在对应AIDL的进程，调起该服务进程；

    3.服务进程实现了aidl存根的Binder引用返回；

```
    @Override
    public IBinder onBind(Intent intent) {
        //实现存根stub中的抽象方法 
        return new IBInderInterface.Stub();
    }
```

    4.客户端进程实现ServiceConnection，该类实现与服务进程通讯的关键；

```
private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //此处的service就是服务进程返回的Binder引用
            IBInderInterface interface = IBInderInterface.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    };
```

在1步时，将ServiceConnection传递给了Binder驱动，此时Binder驱动又将服务进程的Binder引用通过ServiceConnection传递回给

* 客户端进程之所以要拷贝一份与服务端进程相同的aidl信息，其目的:

* 服务端在接收到绑定请求时，在其onBind方法中会返回Binder引用，此时，在客户端进程中能够通过该aidl将Binder引用转化为对应的接口引用；





