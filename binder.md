# Binder驱动

Binder驱动是由系统创建的C++进程，专门用来接收和转发其他进程的通讯信息；

## 原理

Binder驱动会为每一个进程，根据其pid分配一份内存，而每个进程会把自己的aidl一一注册进这份内存中，这份内存也称为共享内存。

在客户端进程需要访问服务端进程时，会将需要访问的服务端的aidl信息传进Binder驱动中（应该是通过**bindService\(intent\)**），Binder驱动根据客户端的访问信息，在内存中查找对应aidl的服务端进程，然后将信息传递给服务端进程。

服务端进程在接收信息之后，会创建一个Binder，并通过Binder驱动传递回来，最终交给了客户端进程。

1.客户端向Binder驱动发出请求，与Binder驱动建立链接：

```
Intent intent = new Intent();
intent.setPackage("com.example.bindertest.aidl.IBInderInterface");
bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
```

2.Binder驱动中找到存在对应AIDL的进程，调起该服务进程；

3.服务进程实现了对应接口的方法，并把对应的Binder引用返回：

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

在1步时，将ServiceConnection传递给了Binder驱动，此时Binder驱动又将服务进程的Binder引用通过ServiceConnection传递回给客户端；

客户端进程之所以要拷贝一份与服务端进程相同的aidl信息，其目的:

* 服务端在接收到绑定请求时，在其onBind方法中会返回Binder引用，此时，在客户端进程中能够通过该aidl将Binder引用转化为对应的接口引用，这样就能调用对应的接口方法；

* 通过IBInderInterface.Stub.asInterface\(service\);返回的其实是代理Proxy的对象，Proxy实际上是stub的  
  内部类，其中又包含了服务进程的Binder引用：

```
private static class Proxy implements com.example.bindertest.aidl.IBInderInterface {
    //mRemote对应的就是Stub的引用
    private android.os.IBinder mRemote;

    Proxy(android.os.IBinder remote) {
        mRemote = remote;
    }

    @Override
    public android.os.IBinder asBinder() {
        return mRemote;
    }

    public java.lang.String getInterfaceDescriptor() {
        return DESCRIPTOR;
    }

    @Override
    public void test(int param) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeInt(param);
            mRemote.transact(Stub.TRANSACTION_test, _data, _reply, 0);
            _reply.readException();
        } finally {
            _reply.recycle();
            _data.recycle();
        }
    }

    @Override
    public int test2(int param) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        int _result;
        try {
            //向Binder驱动确认调用的aidl
            _data.writeInterfaceToken(DESCRIPTOR);
            _data.writeInt(param);
            mRemote.transact(Stub.TRANSACTION_test2, _data, _reply, 0);
            _reply.readException();
            _result = _reply.readInt();
        } finally {
            _reply.recycle();
            _data.recycle();
        }
        return _result;
    }
}
```

在Proxy中，实现了我们能够调用的服务端的接口方法，但其实际是用来把客户端传递的参数存进Parcel中，然后调用  Stub的transact方法来实现向服务端写数据和读取返回值：

```
mRemote.transact(Stub.TRANSACTION_test2, _data, _reply, 0);
```

在服务端进程中，实现了存根stub的实例，包含了对应的接口方法的具体实现逻辑；

当客户端调用transact方法之后，服务端存根的onTransact方法会被Binder驱动调用，客户端的参数在这里被接收；  
 （在客户端调用Proxy的transact方法后，会将要调用方法的数据转化为常量，传递给Binder驱动；Binder驱动根据DESCRIPTOR找到对应内存中的进程PID，根据内存中的信息找到对应服务端，调用onTransact方法，调起对应的方法；）

```
     @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_test: {
                //验证是否是目标aidl
                data.enforceInterface(DESCRIPTOR);
                int _arg0;
                //读取客户端传递的参数
                _arg0 = data.readInt();
                //调用服务端本地实现的test方法
                this.test(_arg0);
                //如果有异常，写回去
                reply.writeNoException();
                return true;
            }
            case TRANSACTION_test2: {
                data.enforceInterface(DESCRIPTOR);
                int _arg0;
                _arg0 = data.readInt();
                int _result = this.test2(_arg0);
                reply.writeNoException();
                //向客户端写返回值
                reply.writeInt(_result);
                return true;
            }
        }
        return super.onTransact(code, data, reply, flags);
    }
```

可以看到，当服务端操作完成后，返回了true，操作结束；此时客户端由于调用transact方法而进入的等待操作也会结束，如果有返回值，通过Parcel（\_reply）就能直接拿到；

_说明：虽然整个流程涉及了进程间操作，但是在客户端进程中，并不是表现为异步的，因为当客户端进程调用服务端方法（通过transact方法），此时客户端直接进入了等待，直到服务端响应，才会继续往下执行。这个流程在客户端上表现为同步！_

