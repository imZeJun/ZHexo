---
title: Android 数据存储知识梳理(1) - SQLiteOpenHelper 源码解析
date: 2017-03-06 17:15
categories : Android 数据存储知识梳理
---
# 一、概述
这篇文章主要涉及到项目当中，使用数据库相关的操作：
- 使用`SQLiteOpenHelper`来封装数据库。
- 多线程的情况下使用`SQLiteOpenHelper`。

# 二、使用`SQLiteOpenHelper`封装数据库
## 2.1 使用`SQLiteOpenHelper`的原因
之所以需要使用`SQLiteOpenHelper`，而不是调用`Context`的方法来直接得到`SQLiteDatabase`，主要是因为它有两个好处：
- **自动管理创建**：当需要对数据库进行操作的时候，不用关心`SQLiteOpenHelper`所关联的`SQLiteDatabase`是否创建，`SQLiteOpenHelper`会帮我们去判断，如果没有创建，那么就先创建该数据库后，再返回给使用者。
- **自动管理版本**：当需要对数据库进行操作之前，如果发现当前声明的数据库的版本和手机内的数据库版本不同的时候，那么会分别调用`onUpgrade`和`onDowngrade`，这样使用者就可以在里面来处理新旧版本的兼容问题。

## 2.2 `SQLiteOpenHelper`的`API`
`SQLiteOpenHelper`的`API`很少，我们来看一下：
![](http://upload-images.jianshu.io/upload_images/1949836-fbc257df47d8e665.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 2.2.1 构造函数
```
    /**
     * Create a helper object to create, open, and/or manage a database.
     * The database is not actually created or opened until one of
     * {@link #getWritableDatabase} or {@link #getReadableDatabase} is called.
     *
     * <p>Accepts input param: a concrete instance of {@link DatabaseErrorHandler} to be
     * used to handle corruption when sqlite reports database corruption.</p>
     *
     * @param context to use to open or create the database
     * @param name of the database file, or null for an in-memory database
     * @param factory to use for creating cursor objects, or null for the default
     * @param version number of the database (starting at 1); if the database is older,
     *     {@link #onUpgrade} will be used to upgrade the database; if the database is
     *     newer, {@link #onDowngrade} will be used to downgrade the database
     * @param errorHandler the {@link DatabaseErrorHandler} to be used when sqlite reports database
     * corruption, or null to use the default error handler.
     */
    public SQLiteOpenHelper(Context context, String name, CursorFactory factory, int version,
            DatabaseErrorHandler errorHandler) {
        if (version < 1) throw new IllegalArgumentException("Version must be >= 1, was " + version);

        mContext = context;
        mName = name;
        mFactory = factory;
        mNewVersion = version;
        mErrorHandler = errorHandler;
    }
```
这里有一点很重要：**当我们实例化一个`SQLiteOpenHelper`的子类时，并不会立刻创建或者打开它对应的数据库，这个操作是等到调用了`getWritableDatabase`或者`getReadableDatabase`才进行的**。
- `context`：用来打开或者关闭数据的上下文，需要注意内存泄露问题。
- `name`：数据库的名字，一般为`xxx.db`，如果为空，那么使用的是内存数据库。
- `factory`：创建`cursor`的工厂类，如果为空，那么使用默认的。
- `version`：数据库的当前版本号，必须大于等于`1`。
- `erroeHandler`：数据库发生错误时的处理者，如果为空，那么使用默认处理方式。

## 2.2.2 获得`SQLiteDatabase`
一般情况下，当我们实例完一个`SQLiteOpenHelper`对象之后，就可以通过它所关联的`SQLiteDatabase`，来对数据库进行操作了，获得数据库的方式有下面两种：
```
    /**
     * Create and/or open a database that will be used for reading and writing.
     * The first time this is called, the database will be opened and
     * {@link #onCreate}, {@link #onUpgrade} and/or {@link #onOpen} will be
     * called.
     *
     * <p>Once opened successfully, the database is cached, so you can
     * call this method every time you need to write to the database.
     * (Make sure to call {@link #close} when you no longer need the database.)
     * Errors such as bad permissions or a full disk may cause this method
     * to fail, but future attempts may succeed if the problem is fixed.</p>
     *
     * <p class="caution">Database upgrade may take a long time, you
     * should not call this method from the application main thread, including
     * from {@link android.content.ContentProvider#onCreate ContentProvider.onCreate()}.
     *
     * @throws SQLiteException if the database cannot be opened for writing
     * @return a read/write database object valid until {@link #close} is called
     */
    public SQLiteDatabase getWritableDatabase() {
        synchronized (this) {
            return getDatabaseLocked(true);
        }
    }

    /**
     * Create and/or open a database.  This will be the same object returned by
     * {@link #getWritableDatabase} unless some problem, such as a full disk,
     * requires the database to be opened read-only.  In that case, a read-only
     * database object will be returned.  If the problem is fixed, a future call
     * to {@link #getWritableDatabase} may succeed, in which case the read-only
     * database object will be closed and the read/write object will be returned
     * in the future.
     *
     * <p class="caution">Like {@link #getWritableDatabase}, this method may
     * take a long time to return, so you should not call it from the
     * application main thread, including from
     * {@link android.content.ContentProvider#onCreate ContentProvider.onCreate()}.
     *
     * @throws SQLiteException if the database cannot be opened
     * @return a database object valid until {@link #getWritableDatabase}
     *     or {@link #close} is called.
     */
    public SQLiteDatabase c() {
        synchronized (this) {
            return getDatabaseLocked(false);
        }
    }
```
注意到，它们最终都是调用了同一个方法，并且在该方法上加上了同步代码块。

关于`getWritableDatabase`，源码当中提到了以下几点：
- 该方法是用来创建或者打开一个**可读写的数据库**，当这个方法第一次被调用时，数据库会被打开，并且`onCreate`，`onUpgrade`或者`onOpen`方法可能会被调用。
- 一旦打开成功之后，这个**数据库会被缓存**，也就是其中的`mDatabase`成员变量，但是如果权限检查失败或者磁盘慢了，那么有可能会打开失败。
- `Upgrade`方法有时候可能会执行耗时的操作，因此**不要在主线程当中调用这个方法，包括`ContentProvider`的`onCreate()`方法**。

关于`getWritableDatabase`，有几点说明：
- 在除了某些特殊情况，它和`getWritableDatabase`返回的一样，都是一个可读写的数据库，如果磁盘满了，那么才有可能返回一个只读的数据库。
- 如果当前`mDatabase`是只读的，但是之后又调用了一个`getWritableDatabase`方法并且成功地获取到了可写的数据库，那么原来的`mDatabase`会被关闭，重新打开一个可读写的数据库，调用`db.reopenReadWrite()`方法。

下面，我们来看一下`getDatabaseLocked`的具体实现，来了解其中的细节问题：
```
    private SQLiteDatabase getDatabaseLocked(boolean writable) {
        if (mDatabase != null) {
            if (!mDatabase.isOpen()) {
                //如果使用者获取了db对象，但不是通过SQLiteOpenHelper关闭它，那么下次调用的时候会返回null。
                mDatabase = null;
            } else if (!writable || !mDatabase.isReadOnly()) {
                //如果不要求可写或者当前缓存的数据库已经是可写的了，那么直接返回.
                return mDatabase;
            }
        }

        if (mIsInitializing) {
            throw new IllegalStateException("getDatabase called recursively");
        }

        SQLiteDatabase db = mDatabase;
        try {
            mIsInitializing = true;
            //如果要求可写，但是当前缓存的是只读的，那么尝试关闭后再重新打开来获取一个可写的。
            if (db != null) {
                if (writable && db.isReadOnly()) {
                    db.reopenReadWrite();
                }
            //下面就是没有缓存的情况.
            } else if (mName == null) {
                db = SQLiteDatabase.create(null);
            //这里就是我们第一次调用时候的情况.
            } else {
                try {
                    if (DEBUG_STRICT_READONLY && !writable) {
                        final String path = mContext.getDatabasePath(mName).getPath();
                        db = SQLiteDatabase.openDatabase(path, mFactory,
                                SQLiteDatabase.OPEN_READONLY, mErrorHandler);
                    } else {
                        //以可写的方式打开或者创建一个数据库，注意这里有一个标志位mEnableWriteAheadLogging，我们后面来解释.
                        db = mContext.openOrCreateDatabase(mName, mEnableWriteAheadLogging ?
                                Context.MODE_ENABLE_WRITE_AHEAD_LOGGING : 0,
                                mFactory, mErrorHandler);
                    }
                } catch (SQLiteException ex) {
                    //如果发生异常，并且要求可写的，那么直接抛出异常.
                    if (writable) {
                        throw ex;
                    }
                    Log.e(TAG, "Couldn't open " + mName
                            + " for writing (will try read-only):", ex);
                    //如果不要求可写，那么尝试调用只读的方式来打开。
                    final String path = mContext.getDatabasePath(mName).getPath();
                    db = SQLiteDatabase.openDatabase(path, mFactory,
                            SQLiteDatabase.OPEN_READONLY, mErrorHandler);
                }
            }
            //抽象方法，子类实现。
            onConfigure(db);
           
            final int version = db.getVersion();
            //如果新旧版本不想等，那么才会进入下面的判断.
            if (version != mNewVersion) {
                //当前数据库是只读的，那么会抛出异常。
                if (db.isReadOnly()) {
                    throw new SQLiteException("Can't upgrade read-only database from version " +
                            db.getVersion() + " to " + mNewVersion + ": " + mName);
                }
                //开启事务，onCreate/onDowngrade/OnUpgrade只会调用其中一个。
                db.beginTransaction();
                try {
                    if (version == 0) {
                        onCreate(db);
                    } else {
                        if (version > mNewVersion) {
                            onDowngrade(db, version, mNewVersion);
                        } else {
                            onUpgrade(db, version, mNewVersion);
                        }
                    }
                    db.setVersion(mNewVersion);
                    db.setTransactionSuccessful();
                } finally {
                    db.endTransaction();
                }
            }
            //数据库打开完毕.
            onOpen(db);

            if (db.isReadOnly()) {
                Log.w(TAG, "Opened " + mName + " in read-only mode");
            }

            mDatabase = db;
            return db;
        } finally {
            mIsInitializing = false;
            if (db != null && db != mDatabase) {
                db.close();
            }
        }
    }
```
### 2.2.3 `onConfig/onOpen`
在上面获取数据库的过程中，有两个方法：
```
    /**
     * Called when the database connection is being configured, to enable features
     * such as write-ahead logging or foreign key support.
     * <p>
     * This method is called before {@link #onCreate}, {@link #onUpgrade},
     * {@link #onDowngrade}, or {@link #onOpen} are called.  It should not modify
     * the database except to configure the database connection as required.
     * </p><p>
     * This method should only call methods that configure the parameters of the
     * database connection, such as {@link SQLiteDatabase#enableWriteAheadLogging}
     * {@link SQLiteDatabase#setForeignKeyConstraintsEnabled},
     * {@link SQLiteDatabase#setLocale}, {@link SQLiteDatabase#setMaximumSize},
     * or executing PRAGMA statements.
     * </p>
     *
     * @param db The database.
     */
    public void onConfigure(SQLiteDatabase db) {}
    /**
     * Called when the database has been opened.  The implementation
     * should check {@link SQLiteDatabase#isReadOnly} before updating the
     * database.
     * <p>
     * This method is called after the database connection has been configured
     * and after the database schema has been created, upgraded or downgraded as necessary.
     * If the database connection must be configured in some way before the schema
     * is created, upgraded, or downgraded, do it in {@link #onConfigure} instead.
     * </p>
     *
     * @param db The database.
     */
    public void onOpen(SQLiteDatabase db) {}
```
- `onConfigure`：在`onCreate/onUpgrade/onDowngrade`调用之前，可以在它其中来配置数据库连接的参数，这时候数据库已经创建完成，但是表有可能还没创建，或者不是最新的。
- `onOpen`：在数据库连接配置完成，并且数据库表已经更新到最新的，当我们在这里对数据库进行操作时，需要判断它是否是只读的。

### 2.2.4 `onCreate/onUpgrade/onDowngrade`
```
    /**
     * Called when the database is created for the first time. This is where the
     * creation of tables and the initial population of the tables should happen.
     *
     * @param db The database.
     */
    public abstract void onCreate(SQLiteDatabase db);

    /**
     * Called when the database needs to be upgraded. The implementation
     * should use this method to drop tables, add tables, or do anything else it
     * needs to upgrade to the new schema version.
     *
     * <p>
     * The SQLite ALTER TABLE documentation can be found
     * <a href="http://sqlite.org/lang_altertable.html">here</a>. If you add new columns
     * you can use ALTER TABLE to insert them into a live table. If you rename or remove columns
     * you can use ALTER TABLE to rename the old table, then create the new table and then
     * populate the new table with the contents of the old table.
     * </p><p>
     * This method executes within a transaction.  If an exception is thrown, all changes
     * will automatically be rolled back.
     * </p>
     *
     * @param db The database.
     * @param oldVersion The old database version.
     * @param newVersion The new database version.
     */
    public abstract void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion);

    /**
     * Called when the database needs to be downgraded. This is strictly similar to
     * {@link #onUpgrade} method, but is called whenever current version is newer than requested one.
     * However, this method is not abstract, so it is not mandatory for a customer to
     * implement it. If not overridden, default implementation will reject downgrade and
     * throws SQLiteException
     *
     * <p>
     * This method executes within a transaction.  If an exception is thrown, all changes
     * will automatically be rolled back.
     * </p>
     *
     * @param db The database.
     * @param oldVersion The old database version.
     * @param newVersion The new database version.
     */
    public void onDowngrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        throw new SQLiteException("Can't downgrade database from version " +
                oldVersion + " to " + newVersion);
    }
```
- `onCreate`原有数据库版本为`0`时调用，在里面我们进行剪标操作；而`onUpgrade/onDowngrade`则在不相等时调用，在里面我们对表的字段进行更改。
- `onDowngrade`的默认实现是抛出异常。
- `onUpgrade`没有默认实现。
- 这三个操作都是放在事务当中，如果发生了错误，那么会回滚。

### 2.2.5 关闭
```
    /**
     * Close any open database object.
     */
    public synchronized void close() {
        if (mIsInitializing) throw new IllegalStateException("Closed during initialization");

        if (mDatabase != null && mDatabase.isOpen()) {
            mDatabase.close();
            mDatabase = null;
        }
    }
```
会关闭当前缓存的数据库，并把清空`mDatabase`缓存，注意这个方法也被加上了对象锁。
# 三、多线程情况下对`SQLiterDBHelper`的使用
- 在多线程的情况下，每个线程对同一个`SQLiterDBHelper`实例进行操作，并不会产生影响，因为刚刚我们看到，在获取和关闭数据库的方法上，都加上了对象锁，所以最终我们只是打开了一条到数据库上的连接，这时候就转变为去讨论`SQLiteDatabase`的增删改查操作是否是线程安全的了。
- 然而，如果每个线程获取`SQLiteDatabase`时，不是用的同一个`SQLiterDBHelper`，那么其实是打开了多个连接，假如通过这多个连接同数据库的操作是没有同步的话，那么就会出现问题。

下面，我们总结一下在多线程情况下，可能出现问题的几种场景：
## 3.1 多线程情况下每个线程创建一个`SQLiteOpenHelper`，并且之前没有创建过关联的`db`
```
    /**
     * 多线程同时创建,每个线程持有一个SQLiteOpenHelper
     * @param view
     */
    public void multiOnCreate(View view) {
        int threadCount = 50;
        for (int i = 0; i < threadCount; i++) {
            Thread thread = new Thread() {
                @Override
                public void run() {
                    MultiThreadDBHelper dbHelper = new MultiThreadDBHelper(MainActivity.this);
                    SQLiteDatabase database = dbHelper.getWritableDatabase();
                    ContentValues contentValues = new ContentValues(1);
                    contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_KEY, "thread_id");
                    contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_VALUE, String.valueOf(Thread.currentThread().getId()));
                    database.insert(MultiThreadDBContract.TABLE_KEY_VALUE.TABLE_NAME, null, contentValues);
                }
            };
            thread.start();
        }
    }
```
在上面这种情况下，由于多个线程的`getWritableDatabase`没有进行同步操作，并且这时候手机里面没有对应的数据库，那么就有可能出现下面的情况：
- `Thread#1`调用`getWritableDatabase`，在其中获取数据库的版本号为`0`，因此它调用`onCreate`建表，建表完成。
- `Thread#1`建表完成，但是还没有来得及给数据库设置版本号时，`Thread#2`也调用了`getWritableDatabase`，在其中它获取数据库版本号也是`0`，因此也执行了`onCreate`操作，那么这时候就会出现对一个数据库多次建立同一张表的情况发生。
![](http://upload-images.jianshu.io/upload_images/1949836-dbdda774931521d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.2 多线程情况下每个线程创建一个`SQLiteOpenHelper`，同时对关联的`db`进行写入操作
```
    /**
     * 多个线程同时写入,每个线程持有一个SQLiteOpenHelper
     * @param view
     */
    public void multiWriteUseMultiDBHelper(View view) {
        MultiThreadDBHelper init = new MultiThreadDBHelper(MainActivity.this);
        SQLiteDatabase database = init.getWritableDatabase();
        database.close();
        int threadCount = 10;
        for (int i = 0; i < threadCount; i++) {
            Thread thread = new Thread() {
                @Override
                public void run() {
                    MultiThreadDBHelper dbHelper = new MultiThreadDBHelper(MainActivity.this);
                    SQLiteDatabase database = dbHelper.getWritableDatabase();
                    for (int i = 0; i < 1000; i++) {
                        ContentValues contentValues = new ContentValues(1);
                        contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_KEY, "thread_id");
                        contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_VALUE, String.valueOf(Thread.currentThread().getId()) + "_" + i);
                        database.insert(MultiThreadDBContract.TABLE_KEY_VALUE.TABLE_NAME, null, contentValues);
                    }
                }
            };
            thread.start();
        }
    }
```
假如我们启动了多个线程，并且在每个线程中新建了`SQLiteOpenHelper`实例，那么当它们调用各自的`getWritableDatabase`方法时，其实是对手机中的`db`建立了多个数据库连接，当通过多个数据库连接同时对`db`进行写入，那么会抛出下面的异常：
![](http://upload-images.jianshu.io/upload_images/1949836-d785c06b6d1251d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从`3.1`和`3.2`我们就可以看出，在多线程的情况下，每个线程新建一个`SQLiteOpenHelper`会出现问题，因此，我们尽量把它设计为单例的模式，那么是不是多个线程持有同一个`SQLiteOpenHelper`实例就不会出现问题呢，其实并不然，我们看一下下面这些共用同一个`SQLiteOpenHelper`的情形。
## 3.3 多线程情况下所有线程共用一个`SQLiteOpenHelper`，其中一个线程调用了`close`方法
```
    /**
     * 多线程下共用一个SQLiteOpenHelper
     * @param view
     */
    public void multiCloseUseOneDBHelper(View view) {
        final MultiThreadDBHelper init = new MultiThreadDBHelper(MainActivity.this);
        final SQLiteDatabase database = init.getWritableDatabase();
        database.close();
        Thread thread1 = new Thread() {

            @Override
            public void run() {
                SQLiteDatabase database = init.getWritableDatabase();
                try {
                    Thread.sleep(1000);
                } catch (Exception e) {
                    Log.e("MainActivity", "e=" + e);
                }
                ContentValues contentValues = new Conten;
                contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_KEY, "thread_id");
                contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_VALUE, String.valueOf(Thread.currentThread().getId()));
                //由于Thread2已经关闭了数据库，因此这里再调用插入操作就会出现问题。
                database.insert(MultiThreadDBContract.TABLE_KEY_VALUE.TABLE_NAME, null, contentValues);
            }
        };
        thread1.start();
        Thread thread2 = new Thread() {
            @Override
            public void run() {
                try {
                    Thread.sleep(100);
                } catch (Exception e) {
                    Log.e("MainActivity", "e=" + e);
                }
                init.close();
            }
        };
        thread2.start();
    }
```
![](http://upload-images.jianshu.io/upload_images/1949836-96841e89679da5c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 3.4 多线程情况下所有线程共用一个`SQLiteOpenHelper`，在写的过程中同时读
由于是共用了同一个`SQLiteOpenHelper`，因此我们需要考虑的是对于同一个`SQLiteDatabase`连接，是否允许读写并发，默认情况下是不允许的，但是，我们可以通过`SQLiteOpenHelper#setWriteAheadLoggingEnabled`，这个配置默认是关的，当开启时表示：它允许一个写线程与多个读线程同时在一个`SQLiteDatabase`上起作用。实现原理是写操作其实是在一个单独的文件，不是原数据库文件。所以写在执行时，不会影响读操作，读操作读的是原数据文件，是写操作开始之前的内容。在写操作执行成功后，会把修改合并会原数据库文件。此时读操作才能读到修改后的内容。但是这样将花费更多的内存。
# 四、解决多线程的例子
工厂类负责根据`dbName`创建对应的`SQLiteOpenHelper`类
```
public abstract class DBHelperFactory {
    public abstract SQLiteOpenHelper createDBHelper(String dbName);
}
```
通过管理类来插入指定**数据库**的**指定表**。
```
public class DBHelperManager {

    private HashMap<String, SQLiteOpenHelperWrapper> mDBHelperWrappers;
    private DBHelperFactory mDBHelperFactory;

    static class Nested {
        public static DBHelperManager sInstance = new DBHelperManager();
    }

    public static DBHelperManager getInstance() {
        return Nested.sInstance;
    }

    private DBHelperManager() {
        mDBHelperWrappers = new HashMap<>();
    }

    public void setDBHelperFactory(DBHelperFactory dbHelperFactory) {
        mDBHelperFactory = dbHelperFactory;
    }

    private synchronized SQLiteOpenHelperWrapper getSQLiteDBHelperWrapper(String dbName) {
        SQLiteOpenHelperWrapper wrapper = mDBHelperWrappers.get(dbName);
        if (wrapper == null) {
            if (mDBHelperFactory != null) {
                SQLiteOpenHelper dbHelper = mDBHelperFactory.createDBHelper(dbName);
                if (dbHelper != null) {
                    SQLiteOpenHelperWrapper newWrapper = new SQLiteOpenHelperWrapper();
                    newWrapper.mSQLiteOpenHelper = dbHelper;
                    newWrapper.mSQLiteOpenHelper.setWriteAheadLoggingEnabled(true);
                    mDBHelperWrappers.put(dbName, newWrapper);
                    wrapper = newWrapper;
                }
            }
        }
        return wrapper;
    }

    private synchronized SQLiteDatabase getReadableDatabase(String dbName) {
        SQLiteOpenHelperWrapper wrapper = getSQLiteDBHelperWrapper(dbName);
        if (wrapper != null && wrapper.mSQLiteOpenHelper != null) {
            return wrapper.mSQLiteOpenHelper.getReadableDatabase();
        } else {
            return null;
        }
    }

    private synchronized SQLiteDatabase getWritableDatabase(String dbName) {
        SQLiteOpenHelperWrapper wrapper = getSQLiteDBHelperWrapper(dbName);
        if (wrapper != null && wrapper.mSQLiteOpenHelper != null) {
            return wrapper.mSQLiteOpenHelper.getWritableDatabase();
        } else {
            return null;
        }
    }

    private class SQLiteOpenHelperWrapper {
        public SQLiteOpenHelper mSQLiteOpenHelper;
    }

    public long insert(String dbName, String tableName, String nullColumn, ContentValues contentValues) {
        SQLiteDatabase db = getWritableDatabase(dbName);
        if (db != null) {
            return db.insert(tableName, nullColumn, contentValues);
        }
        return -1;
    }

    public Cursor query(String dbName, String table, String[] columns, String selection, String[] selectionArgs, String groupBy, String having, String orderBy) {
        SQLiteDatabase db = getReadableDatabase(dbName);
        if (db != null) {
            return db.query(table, columns, selection, selectionArgs, groupBy, having, orderBy);
        }
        return null;
    }

    public int update(String dbName, String table, ContentValues values, String whereClause, String[] whereArgs) {
        SQLiteDatabase db = getWritableDatabase(dbName);
        if (db != null) {
            return db.update(table, values, whereClause, whereArgs);
        }
        return 0;
    }

    public int delete(String dbName, String table, String whereClause, String[] whereArgs) {
        SQLiteDatabase db = getWritableDatabase(dbName);
        if (db != null) {
            return db.delete(table, whereClause, whereArgs);
        }
        return 0;
    }

}
```
多线程插入的方式改为下面这样：
```
    public void multiWriteUseManager(View view) {
        int threadCount = 10;
        for (int i = 0; i < threadCount; i++) {
            Thread thread = new Thread() {
                @Override
                public void run() {
                    for (int i = 0; i < 1000; i++) {
                        ContentValues contentValues = new ContentValues(1);
                        contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_KEY, "thread_id");
                        contentValues.put(MultiThreadDBContract.TABLE_KEY_VALUE.COLUMN_VALUE, String.valueOf(Thread.currentThread().getId()) + "_" + i);
                        DBHelperManager.getInstance().insert(MultiThreadDBContract.DATABASE_NAME, MultiThreadDBContract.TABLE_KEY_VALUE.TABLE_NAME, null, contentValues);
                    }
                }
            };
            thread.start();
        }
    }
```
# 五、小结
这篇文章主要介绍的是`SQLiteOpenHelper`，需要注意以下三点：
- 不要在多线程情况且没有进行线程同步的情况下，操作由不同的`SQLiteOpenHelper`对象所返回的`SQLiteDatabase`。
- 在多线程共用一个`SQLiteOpenHelper`时，需要注意关闭时，是否有其它线程正在使用该`Helper`所关联的`db`。
- 在多线程共用一个`SQLiteOpenHelper`时，是否有同时读写的需求，如果有，那么需要设置`setWriteAheadLoggingEnabled`标志位。

对于`SQLiteDatabase`，还有更多的优化操作，当我们有关数据库的错误时，我们都可以根据错误码，在下面的网站当中找到说明：
> [`https://www.sqlite.org/rescode.html`](https://www.sqlite.org/rescode.html)
