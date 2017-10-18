[TOC]

## 同一应用的Activity之间数据传输是进程间通讯

对于系统服务而言，不同应用中的Activity是没有区别的，也就是说，系统服务不管你是哪个应用的页面，其中维护一个栈，用来记录当面和后面将要进行显示的Activity。

对于系统服务（AMS）而言，其只管理当前Activity需要进行的生命周期活动，所有的Activity都会被其内部维护的栈进行管理。

在AMS中，会维护一个进程栈，该栈中存放的是每一个进程的Activity栈，后启动的应用，其Activity栈将会在上面。当Activity栈中的所有Activity退出之后，进程才会从进程栈中退出。

**tips：**由于Activity栈是在远程系统服务中被维护，所有我们在自己的应用中是无法拿到Activity栈的，只能通过自定义一个ArrayList的Activity栈，来进行维护；

## 进程间通讯原理与类似于CS架构

1、类似于服务端-客户端结构，它们之间的内存是相互隔离的，之间的通讯只能通过IPC进行；

2、可以把Binder驱动认为是两个进程间通讯的通讯服务器，类似于消息中转站的功能；

3、不同于把两个进程当做服务器和客户端，两个进程是存在于同于应用层次，而Binder驱动位于系统底部的C++层，并且Binder驱动作为消息中转的服务器，不同于服务进程，Binder驱动不对消息进行处理，只做传递。而服务进程是要对客户端进程的操作进行相应的处理和反馈的。

//TODO 图解

## AIDL：Android Interface Define Language

aidl即安卓接口定义语言，是一套存在于客户端，服务端，定义内容完全一致的通讯语言；
其原理是，客户端在本地对aidl进行操作，形成对操作行为的描述，这套描述会经过**Binder驱动**，最终传递给了服务端。
服务端通过解析这套行为描述，最终将操作映射到自己的aidl中，这样服务端和客户端之间的操作就能够同步进行；

### 存根（stub）
> 存根：存根类是一个类，它实现了一个接口，但是实现后的每个方法都是空的。
它的作用是：如果一个接口有很多方法，如果要实现这个接口，就要实现所有的方法。但是一个类从业务来说，可能只需要其中一两个方法。如果直接去实现这个接口，除了实现所需的方法，还要实现其他所有的无关方法。而如果通过继承存根类就实现接口，就免去了这种麻烦。
类似于一些SimpleXXX类。

存根的作用是从Binder驱动中接收数据；


### 代理（Proxy）

代理的作用是将本地操作解析为特定数据，并向Binder驱动中写入；

    // 1
    public interface IBInderInterface extends android.os.IInterface {
        /**
         * Local-side IPC implementation stub class.
         */
        public static abstract class Stub extends android.os.Binder implements com.example.bindertest.aidl.IBInderInterface {
            // 2
            private static final java.lang.String DESCRIPTOR = "com.example.bindertest.aidl.IBInderInterface";
    
            /**
             * Construct the stub at attach it to the interface.
             */
            public Stub() {
                this.attachInterface(this, DESCRIPTOR);
            }
    
            /**
             * Cast an IBinder object into an com.example.bindertest.aidl.IBInderInterface interface,
             * generating a proxy if needed.
             */
            public static com.example.bindertest.aidl.IBInderInterface asInterface(android.os.IBinder obj) {
                if ((obj == null)) {
                    return null;
                }
                android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
                if (((iin != null) && (iin instanceof com.example.bindertest.aidl.IBInderInterface))) {
                    return ((com.example.bindertest.aidl.IBInderInterface) iin);
                }
                return new com.example.bindertest.aidl.IBInderInterface.Stub.Proxy(obj);
            }
    
            @Override
            public android.os.IBinder asBinder() {
                return this;
            }
    
            @Override
            public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
                switch (code) {
                    case INTERFACE_TRANSACTION: {
                        reply.writeString(DESCRIPTOR);
                        return true;
                    }
                    case TRANSACTION_test: {
                        data.enforceInterface(DESCRIPTOR);
                        int _arg0;
                        _arg0 = data.readInt();
                        this.test(_arg0);
                        reply.writeNoException();
                        return true;
                    }
                    case TRANSACTION_test2: {
                        data.enforceInterface(DESCRIPTOR);
                        int _arg0;
                        _arg0 = data.readInt();
                        int _result = this.test2(_arg0);
                        reply.writeNoException();
                        reply.writeInt(_result);
                        return true;
                    }
                }
                return super.onTransact(code, data, reply, flags);
            }
    
            private static class Proxy implements com.example.bindertest.aidl.IBInderInterface {
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
    
            static final int TRANSACTION_test = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
            static final int TRANSACTION_test2 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        }
    
        public void test(int param) throws android.os.RemoteException;
    
        public int test2(int param) throws android.os.RemoteException;
    }

1、所有的aidl接口都需要继承android.os.IInterface接口；该接口是与C++层通讯协议；
2、DESCRIPTOR 的作用：
- 每一个aidl接口的唯一表示，因为进程中可能存在多个aidl接口；
- 在需要进行反射时，直接使用该标识就可以找到对应类；

## Binder驱动位于C++层

Binder驱动位于系统的C++层的独立进程，位于Android系统专门分配出来的内存中，用来实现进程间的通讯。

