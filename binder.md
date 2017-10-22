# Binder驱动

Binder驱动是由系统创建的C++进程，专门用来接收和转发其他进程的通讯信息；

## 原理

Binder驱动会为每一个进程，根据其pid分配一份内存，而每个进程会把自己的aidl一一注册进这份内存中；

在客户端进程需要访问服务端进程时，会将需要访问的服务端的aidl信息传进Binder驱动中（应该是通过**bindService\(intent\)**），Binder驱动根据客户端的访问信息，在内存中查找对应aidl的服务端进程，然后将信息传递给服务端进程。

服务端进程在接收信息之后，会创建一个Binder，并通过Binder驱动传递回来，最终交给了客户端进程。

1.客户端向Binder驱动发出请求，与Binder驱动建立链接：

```
Intent intent = new Intent();
intent.setPackage("com.example.bindertest.aidl.IBInderInterface");
bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
```

2.Binder驱动中找到存在对应AIDL的进程，调起该服务进程；

3.服务进程实现了aidl存根的Binder引用并返回；

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

* 服务端在接收到绑定请求时，在其onBind方法中会返回Binder引用，此时，在客户端进程中能够通过该aidl将Binder引用转化为对应的接口引用，这样就能调用对应的接口方法（**这其实就是存根stub的作用！**）；

* 通过IBInderInterface.Stub.asInterface(service);返回的其实是代理Proxy的对象，Proxy实际上是stub的
内部类，其中又包含了Stub 的引用：


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

 在Proxy中，实现了我们能够调用的服务端的接口方法，但其实际是用来把客户端传递的参数转化为特定格式数据（Parcel），然后调用Stub的transact方法来实现向服务端写数据和读取返回值：
 
    mRemote.transact(Stub.TRANSACTION_test2, _data, _reply, 0);

 其中，传递参数和获取返回值通过：
 
     android.os.Parcel _data = android.os.Parcel.obtain();
     android.os.Parcel _reply = android.os.Parcel.obtain();

 
         @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_test: {
                    //连接
                    data.enforceInterface(DESCRIPTOR);
                    int _arg0;
                    _arg0 = data.readInt();
                    //向服务端写参数
                    this.test(_arg0);
                    //读取返回异常
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
 
 可以看到，最终向服务端写数据时，是调用stub的对应的接口方法：
     
     public void test(int param) throws android.os.RemoteException;
     public int test2(int param) throws android.os.RemoteException;

 这两个方法实际应该是与Binder驱动通讯用的，并没有具体实现，而两个Parcel实际只是在本地通讯用的，用来在stub和proxy之间进行参数传递。


