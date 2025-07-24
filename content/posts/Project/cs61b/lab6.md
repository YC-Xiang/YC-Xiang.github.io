# Java and Compilation

```java
javac capers/Main.java
java capers.Main story "this is a single argument"
```

# File & Directory Manipulation in Java

## File

```java
File f = new File("dummy.txt");
f.createNewFile(); // 创建文件
f.exists() // 检查文件是否存在
Utils.writeContents(f, "Hello World"); // 将字符串写入文件
```

## Directory

```java
File d = new File("dummy");
d.mkdir(); // 创建目录
```

# Serializable

如果有更复杂的 object 需要保存，可以利用 `java.io.Serializable`, 将 object 序列化成 byte stream 保存到文件中：

```java
import java.io.Serializable;

public class Model implements Serializable {
    ...
}
```

把对象序列化成 byte stream：

```java
Model m = ....;
File outFile = new File(saveFileName);
try {
    ObjectOutputStream out =
        new ObjectOutputStream(new FileOutputStream(outFile));
    out.writeObject(m);
    out.close();
} catch (IOException excp) {
    ...
}
```

反序列化对象：

```java
Model m;
File inFile = new File(saveFileName);
try {
    ObjectInputStream inp =
        new ObjectInputStream(new FileInputStream(inFile));
    m = (Model) inp.readObject();
    inp.close();
} catch (IOException | ClassNotFoundException excp) {
    ...
    m = null;
}
```

为了简化操作，在 cs61b capers Utils class 中提供了 `writeObject` 和 `readObject` 方法：

```java
Model m;
File outFile = new File(saveFileName);

// Serializing the Model object
writeObject(outFile, m);

Model m;
File inFile = new File(saveFileName);

// Deserializing the Model object
m = readObject(inFile, Model.class);
```

# Useful Util Functions

```java
// 写 strings/byte arrays 到文件，允许传入任意数量参数
static void writeContents(File file, Object... contents);

// 读取文件内容，返回 String
static String readContentsAsString(File file);

// 读取文件内容，返回 byte array
static byte[] readContents(File file);

// 将 object 序列化到文件
static void writeObject(File file, Serializable obj);

// 从文件中反序列化 object
static <T extends Serializable> T readObject(File file, Class<T> expectedClass);

// 连接多个字符串，返回新的 File 对象
static File join(String first, String... others);
```

# Testing

```shell
make check
```

# Mandatory Epilogue: Debugging

Intellij Run->Run, 选择 Remote JVM Debug

在 Intellij 中打上断点，执行：

```shell
python3 runner.py --debug our/test02-two-part-story.in
python3 runner.py --keep --debug our/test02-two-part-story.in # 保留生成的.caper文件夹
# 注意报错的话，可能是没设置REPO_DIR环境变量
```
