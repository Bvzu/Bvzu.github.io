# 一种键值对读写锁的尝试
博主在实际工作中，遇到这么一种场景：
前台向后台下发指令，后台根据指令执行轮询任务，每条任务之间相互独立。在到达轮询周期时，读取内存中最新的指令，进行任务，同时，前台可能在任意时刻对某条任务的指令数据进行修改，这时，很明显，需要一个读写锁。但是普通的读写锁，需要是全局的，并且锁的时候会把所有的整个数据全锁住，如博主用的map，而这个时候我只是想要对某个键值对的数据进行读写锁保护，所以便有了如下尝试。

## 定义可拷贝锁对象
因为mutex和condition_variable都是不可以拷贝的，所以这里进行简单的封装，通过不断的new和delete进行。
```
#include <mutex>
#include <condition_variable>

class RWLock{
public:
	RWLock() :  readCount(0),
				writeCount(0),
				isWriting(false){
		lock = new std::mutex();
		readCnd = new std::condition_variable();
		writeCnd = new std::condition_variable();
	}

	RWLock(const RWLock& rwlock){
		readCount = rwlock.readCount;
		writeCount = rwlock.writeCount;
		isWriting = rwlock.isWriting;
		readCnd = rwlock.readCnd;
		writeCnd = rwlock.writeCnd;
	}

	RWLock& operator=(const RWLock& rwlock){
		return *this;
	}

	virtual ~RWLock(){
		delete lock;
		delete readCnd;
		delete writeCnd;
	}

	void lockWrite(){
		std::unique_lock<std::mutex> gud(*lock);
		++writeCount;
		writeCnd->wait(gud, [=]{
			return (0 == readCount) && !isWriting;
		});

		isWriting = true;
	}

	void unlockWrite(){
		std::unique_lock<std::mutex> gud(*lock);
		isWriting = false;
		if(0 == (--writeCount)){
			readCnd->notify_all();
		}else{
			writeCnd->notify_one();
		}
	}

	void lockRead(){
		std::unique_lock<std::mutex> gud(*lock);
		readCnd->wait(gud, [=]{
			return 0 == writeCount;
		});
		--readCount;
	}

	void unlockRead(){
		std::unique_lock<std::mutex> gud(*lock);
		readCnd->wait(gud, [=]{
			return 0 == writeCount;
		});
		++readCount;
	}
private:
	volatile int readCount;
	volatile int writeCount;
	volatile bool isWriting;
	std::mutex *lock;
	std::condition_variable *readCnd;
	std::condition_variable *writeCnd;
}
```
# 写锁
```

class WriteGud{
public:
	explicit WriteGud(RWLock& rwlock) : lock(rwlock){
		lock.lockWrite();
	}

	virtual ~WriteGud(){
		lock.unlockWrite();
	}
private:
	WriteGud(const WriteGud&);
	WriteGud& operator=(const WriteGud&);
	RWLock& lock;
}

```
# 读锁
```

class ReadGud{
public:
	explicit ReadGud(RWLock& rwlock) : lock(rwlock){
		lock.lockRead();
	}

	virtual ~ReadGud(){
		lock.unlockRead();
	}
private:
	ReadGud(const ReadGud&);
	ReadGud& operator=(const ReadGud&);
	RWLock& lock;
}

```

# 使用

```
std::map<RWLock, Data> mData;
RWLock rwlock;
Data data;
mData[rwlock] =  data;

int getData(){
	ReadGud readgud(rwlock);
	...
}

int setData(){
	WriteGud writegud(rwlock);
	...
}
```

# 说明
这只是学习过程中的一种尝试，没有进行过测试，如果有问题，请不吝指教，我将万分感谢。
