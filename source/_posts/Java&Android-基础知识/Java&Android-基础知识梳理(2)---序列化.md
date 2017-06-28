---
title: Java&Android 基础知识梳理(2) - 序列化
date: 2017-02-20 20:44
categories : Java&Android 基础知识梳理
---
# 一、`Parcelable`和`Serializable`

对象的序列化是把`Java`对象转化为字节序列并存储至一个存储媒介（硬盘或者内存）的过程，反序列化则是把字节序列恢复为`Java`对象的过程，但它们仅处理`Java`变量而不处理方法。

序列化的原因：
- 永久性保存对象，保存对象的字节序列到本地文件中。`Serializable`
- 通过序列化对象在网络中传递对象。`Serializable`
- 通过序列化在进程间传递对象。`Parcelable`

两种序列化的区别：
- `Serializable`只需要对某个类以及它的属性实现`Serializable`接口即可，它的缺点是使用了反射，序列化的过程比较慢，这种机制会在序列化的时候创建许多的临时对象，容易引发频繁的`gc`。
- 而`Parcelable`是`Android`平台特有的，在使用内存的时候性能更好，但`Parcelable`不能使用在要将数据存储在磁盘的情况下，因为`Parcelable`不能很好的保证数据的持续性在外界有变化的情况。

# 二、序列化在`Android`平台上的应用
## 2.1 通过`intent`传递复杂对象
`intent`支持传递的数据类型包括： 
- 基本类型的数据、及其数组。
- `String/CharSequence`类型的数据、及其数组。
- `Parcelable/Serializable`，及其数组/列表数据。

## 2.2 `SharePreference`存储复杂对象

# 三、`Serializable`和`Parcelable`
## 3.1 使用`Serializable`的读写操作
首先定义我们要序列化的对象。 
```
public class SBook implements Serializable {            
     public int id;    
     public String name;
}
```
进行读写操作：
```
    private void readSerializable() {
        ObjectInputStream object = null;
        try {
            FileInputStream out = new FileInputStream(Environment.getExternalStorageDirectory() + "/sbook.txt");
            object = new ObjectInputStream(out);
            SBook book = (SBook) object.readObject();
            if (book != null) {
                Log.d(TAG, "book, id=" + book.id + ",name=" + book.name);
            } else {
                Log.d(TAG, "book is null");
            }
        } catch (Exception e) {
            Log.d(TAG, "readSerializable:" + e);
        } finally {
            try {
                if (object != null) {
                    object.close();
                }
            } catch (Exception e) {
                Log.d(TAG, "readSerializable:" + e);
            }
        }
    }

    private void writeSerializable() {
        SBook book = new SBook();
        book.id = 1;
        book.name = "SBook";
        ObjectOutputStream object = null;
        try {
            FileOutputStream out = new FileOutputStream(Environment.getExternalStorageDirectory()  + "/sbook.txt");
            object = new ObjectOutputStream(out);
            object.writeObject(book);
            object.flush();
        } catch (Exception e) {
            Log.d(TAG, "writeSerializable:" + e);
        } finally {
            try {
                if (object != null) {
                    object.close();
                }
            } catch (Exception e) {
                Log.d(TAG, "writeSerializable:" + e);
            }
        }
    }
```
## 3.2 使用`Parcelable`的读写操作
定义序列化对象：
```
public class PBook implements Parcelable {

    public int id;
    public String name;

    public PBook(int id, String name) {
        this.id = id;
        this.name = name;
    }

    private PBook(Parcel in) {
        id = in.readInt();
        name = in.readString();
    }

    public static final Parcelable.Creator<PBook> CREATOR = new Parcelable.Creator<PBook>() {

        @Override
        public PBook[] newArray(int size) {
            return new PBook[size];
        }

        @Override
        public PBook createFromParcel(Parcel source) {
            return new PBook(source);
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
    }
}
```
写入和读取：
```
    private Intent writeParcelable() {
        PBook tony = new PBook(1, "tony");
        PBook king = new PBook(2, "king");
        ArrayList<PBook> list = new ArrayList<>();
        list.add(tony);
        list.add(king);
        Intent intent = new Intent();
        intent.putParcelableArrayListExtra("PBook", list);
        return intent;
    }

    private void readParcelable(Intent intent) {
        if (intent != null) {
            ArrayList<PBook> list = intent.getParcelableArrayListExtra("PBook");
            if (list != null) {
                for (PBook book : list) {
                    Log.d(TAG, "readParcelable, id=" + book.id + ", name=" + book.name);
                }
            }
        }
    }
```
## 四、`SharePreference`存储复杂对象
```
    //obejct -> ObjectOutputStream(ByteArrayOutputStream) -> ByteArrayOutputStream() -> byte[] -> String -> sp
    private void writeSP() {
        SBook book = new SBook();
        book.id = 2;
        book.name = "sp";
        SharedPreferences sp = getSharedPreferences("SBookSP", MODE_PRIVATE);
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        try {
            ObjectOutputStream object = new ObjectOutputStream(os);
            object.writeObject(book);
            String base64 = new String(Base64.encode(os.toByteArray(), Base64.DEFAULT));
            SharedPreferences.Editor editor = sp.edit();
            editor.putString("SBook", base64);
            editor.apply();
        } catch (Exception e) {}
    }
    
    //sp -> string -> byte[] -> ByteArrayInputStream(byte[]) -> ObjectInputStream(ByteArrayInputStream) -> object
    private void readSP() {
        SharedPreferences sp = getSharedPreferences("SBookSP", MODE_PRIVATE);
        String sbook = sp.getString("SBook", "");
        if (sbook.length() > 0) {
            byte[] base64 = Base64.decode(sbook.getBytes(), Base64.DEFAULT);
            ByteArrayInputStream is = new ByteArrayInputStream(base64);
            try {
                ObjectInputStream object = new ObjectInputStream(is);
                SBook book = (SBook) object.readObject();
                if (book != null) {
                    Log.d(TAG, "readSP, id=" + book.id + ", name=" + book.name);
                }
            } catch (Exception e) {

            }
        }
    }
```
