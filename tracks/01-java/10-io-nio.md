# 10. I/O, NIO & NIO.2

[← Назад к списку тем](README.md)

---

## Classic I/O (java.io)

### Stream Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                       I/O Stream Hierarchy                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Byte Streams (binary data):           Character Streams (text):   │
│                                                                     │
│  InputStream                           Reader                       │
│  ├── FileInputStream                   ├── FileReader               │
│  ├── ByteArrayInputStream              ├── StringReader             │
│  ├── BufferedInputStream               ├── BufferedReader           │
│  └── DataInputStream                   └── InputStreamReader        │
│                                                                     │
│  OutputStream                          Writer                       │
│  ├── FileOutputStream                  ├── FileWriter               │
│  ├── ByteArrayOutputStream             ├── StringWriter             │
│  ├── BufferedOutputStream              ├── BufferedWriter           │
│  └── DataOutputStream                  └── OutputStreamWriter       │
│                                                                     │
│  Key Points:                                                        │
│  - Byte streams: raw bytes (images, binary files)                   │
│  - Character streams: text with encoding                            │
│  - Always use BufferedXxx for performance                           │
│  - Always close streams (try-with-resources)                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Reading Files

```java
// Byte stream - reading binary data
try (FileInputStream fis = new FileInputStream("image.png")) {
    byte[] buffer = new byte[1024];
    int bytesRead;
    while ((bytesRead = fis.read(buffer)) != -1) {
        // Process buffer[0..bytesRead-1]
    }
}

// Character stream - reading text
try (FileReader reader = new FileReader("file.txt")) {
    char[] buffer = new char[1024];
    int charsRead;
    while ((charsRead = reader.read(buffer)) != -1) {
        // Process buffer[0..charsRead-1]
    }
}

// Buffered reading (always prefer!)
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}

// With encoding
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(
            new FileInputStream("file.txt"),
            StandardCharsets.UTF_8))) {
    // ...
}
```

### Writing Files

```java
// Byte stream
try (FileOutputStream fos = new FileOutputStream("output.bin")) {
    byte[] data = {0x00, 0x01, 0x02};
    fos.write(data);
}

// Character stream
try (FileWriter writer = new FileWriter("output.txt")) {
    writer.write("Hello, World!");
}

// Buffered writing (always prefer!)
try (BufferedWriter writer = new BufferedWriter(new FileWriter("output.txt"))) {
    writer.write("Line 1");
    writer.newLine();
    writer.write("Line 2");
}

// PrintWriter - convenient methods
try (PrintWriter writer = new PrintWriter("output.txt")) {
    writer.println("Line 1");
    writer.printf("Value: %d%n", 42);
}

// Append mode
try (FileWriter writer = new FileWriter("log.txt", true)) {  // true = append
    writer.write("New log entry\n");
}
```

### Data Streams

```java
// DataOutputStream - write primitive types
try (DataOutputStream dos = new DataOutputStream(
        new FileOutputStream("data.bin"))) {
    dos.writeInt(42);
    dos.writeDouble(3.14);
    dos.writeUTF("Hello");
    dos.writeBoolean(true);
}

// DataInputStream - read primitive types
try (DataInputStream dis = new DataInputStream(
        new FileInputStream("data.bin"))) {
    int i = dis.readInt();
    double d = dis.readDouble();
    String s = dis.readUTF();
    boolean b = dis.readBoolean();
}
```

### Object Serialization

```java
// Serializable class
public class Person implements Serializable {
    private static final long serialVersionUID = 1L;

    private String name;
    private int age;
    private transient String password;  // Not serialized

    // getters, setters...
}

// Write object
try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("person.ser"))) {
    Person person = new Person("John", 30);
    oos.writeObject(person);
}

// Read object
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("person.ser"))) {
    Person person = (Person) ois.readObject();
}

// serialVersionUID - version control for serialization
// If not specified, JVM generates based on class structure
// Change it when making incompatible changes
```

---

## NIO (java.nio)

### Buffers

```java
// Buffer - container for data
// Key properties: capacity, position, limit, mark

// Create buffers
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);  // Off-heap
CharBuffer charBuffer = CharBuffer.allocate(1024);

// Wrap existing array
byte[] array = new byte[1024];
ByteBuffer wrapped = ByteBuffer.wrap(array);

// Writing to buffer
buffer.put((byte) 1);
buffer.put(new byte[]{2, 3, 4});
buffer.putInt(42);  // 4 bytes

// Flip for reading (position=0, limit=previous position)
buffer.flip();

// Reading from buffer
byte b = buffer.get();
int i = buffer.getInt();

// Clear for rewriting (position=0, limit=capacity)
buffer.clear();

// Compact - move unread data to beginning
buffer.compact();

// Rewind - re-read data (position=0, limit unchanged)
buffer.rewind();
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Buffer State Diagram                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  After allocate(10):                                                │
│  [_|_|_|_|_|_|_|_|_|_]   position=0, limit=10, capacity=10         │
│   ^                   ^                                             │
│   position            limit                                         │
│                                                                     │
│  After put(1,2,3,4,5):                                              │
│  [1|2|3|4|5|_|_|_|_|_]   position=5, limit=10, capacity=10         │
│             ^         ^                                             │
│             position  limit                                         │
│                                                                     │
│  After flip():                                                      │
│  [1|2|3|4|5|_|_|_|_|_]   position=0, limit=5, capacity=10          │
│   ^         ^                                                       │
│   position  limit                                                   │
│                                                                     │
│  After get() x3:                                                    │
│  [1|2|3|4|5|_|_|_|_|_]   position=3, limit=5, capacity=10          │
│         ^   ^                                                       │
│         pos limit                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Channels

```java
// Channel - bidirectional connection for I/O
// Key types: FileChannel, SocketChannel, ServerSocketChannel, DatagramChannel

// FileChannel - reading
try (FileChannel channel = FileChannel.open(
        Path.of("file.txt"), StandardOpenOption.READ)) {

    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while (channel.read(buffer) > 0) {
        buffer.flip();
        // Process buffer
        buffer.clear();
    }
}

// FileChannel - writing
try (FileChannel channel = FileChannel.open(
        Path.of("output.txt"),
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE)) {

    ByteBuffer buffer = ByteBuffer.wrap("Hello".getBytes());
    channel.write(buffer);
}

// Transfer between channels (efficient!)
try (FileChannel source = FileChannel.open(Path.of("source.txt"), StandardOpenOption.READ);
     FileChannel dest = FileChannel.open(Path.of("dest.txt"),
         StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {

    source.transferTo(0, source.size(), dest);
}

// Memory-mapped file
try (FileChannel channel = FileChannel.open(
        Path.of("largefile.dat"), StandardOpenOption.READ)) {

    MappedByteBuffer mapped = channel.map(
        FileChannel.MapMode.READ_ONLY, 0, channel.size());

    // Access file as if it's in memory
    while (mapped.hasRemaining()) {
        byte b = mapped.get();
    }
}
```

### Selectors (Non-blocking I/O)

```java
// Selector - multiplexed I/O for non-blocking channels

// Server example
Selector selector = Selector.open();

ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();  // Block until events

    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> iter = selectedKeys.iterator();

    while (iter.hasNext()) {
        SelectionKey key = iter.next();
        iter.remove();

        if (key.isAcceptable()) {
            // Accept connection
            SocketChannel client = serverChannel.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        }

        if (key.isReadable()) {
            // Read data
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            int bytesRead = client.read(buffer);
            // Process data
        }
    }
}
```

---

## NIO.2 (java.nio.file) - Java 7+

### Path

```java
// Creating paths
Path path1 = Path.of("file.txt");
Path path2 = Path.of("/home", "user", "file.txt");
Path path3 = Paths.get("file.txt");  // Old way, same result

// Path operations
Path absolute = path1.toAbsolutePath();
Path normalized = path2.normalize();  // Remove . and ..
Path resolved = path1.resolve("subdir/file.txt");  // Combine paths
Path relativized = path1.relativize(path2);  // Get relative path
Path parent = path1.getParent();
Path fileName = path1.getFileName();

// Path info
path1.isAbsolute();
path1.getNameCount();
path1.getName(0);  // First element
path1.startsWith("/home");
path1.endsWith(".txt");

// Convert
File file = path1.toFile();
URI uri = path1.toUri();
```

### Files Utility Class

```java
// Reading files
String content = Files.readString(Path.of("file.txt"));
byte[] bytes = Files.readAllBytes(Path.of("file.bin"));
List<String> lines = Files.readAllLines(Path.of("file.txt"));

// Streaming lines (memory efficient for large files)
try (Stream<String> lines = Files.lines(Path.of("large.txt"))) {
    lines.filter(line -> line.contains("error"))
         .forEach(System.out::println);
}

// Writing files
Files.writeString(Path.of("file.txt"), "content");
Files.write(Path.of("file.bin"), bytes);
Files.write(Path.of("file.txt"), lines);

// Append
Files.writeString(path, "more content", StandardOpenOption.APPEND);

// Copy/Move/Delete
Files.copy(source, target);
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.move(source, target);
Files.delete(path);
Files.deleteIfExists(path);

// Create
Files.createFile(path);
Files.createDirectory(path);
Files.createDirectories(path);  // Create parent dirs too
Files.createTempFile("prefix", ".tmp");
Files.createTempDirectory("prefix");

// Check
Files.exists(path);
Files.notExists(path);
Files.isReadable(path);
Files.isWritable(path);
Files.isDirectory(path);
Files.isRegularFile(path);
Files.isSymbolicLink(path);
Files.size(path);
Files.getLastModifiedTime(path);

// Attributes
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
attrs.creationTime();
attrs.lastModifiedTime();
attrs.size();
attrs.isDirectory();
```

### Directory Operations

```java
// List directory contents
try (DirectoryStream<Path> stream = Files.newDirectoryStream(dir)) {
    for (Path entry : stream) {
        System.out.println(entry);
    }
}

// With filter
try (DirectoryStream<Path> stream = Files.newDirectoryStream(dir, "*.txt")) {
    for (Path entry : stream) {
        System.out.println(entry);
    }
}

// Walk directory tree
try (Stream<Path> walk = Files.walk(dir)) {
    walk.filter(Files::isRegularFile)
        .filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}

// Walk with depth limit
try (Stream<Path> walk = Files.walk(dir, 2)) {  // Max 2 levels deep
    // ...
}

// Find files
try (Stream<Path> found = Files.find(dir, 10,
        (path, attrs) -> attrs.isRegularFile() && path.toString().endsWith(".java"))) {
    found.forEach(System.out::println);
}
```

### WatchService

```java
// Watch directory for changes
WatchService watchService = FileSystems.getDefault().newWatchService();

Path dir = Path.of("/path/to/watch");
dir.register(watchService,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_DELETE,
    StandardWatchEventKinds.ENTRY_MODIFY);

while (true) {
    WatchKey key = watchService.take();  // Blocks

    for (WatchEvent<?> event : key.pollEvents()) {
        WatchEvent.Kind<?> kind = event.kind();

        if (kind == StandardWatchEventKinds.OVERFLOW) {
            continue;
        }

        WatchEvent<Path> ev = (WatchEvent<Path>) event;
        Path filename = ev.context();

        System.out.println(kind + ": " + filename);
    }

    boolean valid = key.reset();
    if (!valid) {
        break;
    }
}
```

---

## Comparing I/O Approaches

```
┌─────────────────────────────────────────────────────────────────────┐
│                    I/O Comparison                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Classic I/O (java.io):                                             │
│  + Simple API                                                       │
│  + Good for small files                                             │
│  - Blocking only                                                    │
│  - Stream-based (not seekable)                                      │
│                                                                     │
│  NIO (java.nio):                                                    │
│  + Non-blocking possible                                            │
│  + Buffer-based (seekable)                                          │
│  + Memory-mapped files                                              │
│  + Channel transfers                                                │
│  - More complex API                                                 │
│                                                                     │
│  NIO.2 (java.nio.file):                                             │
│  + Modern file operations                                           │
│  + Path API (better than File)                                      │
│  + Files utility class                                              │
│  + Watch service                                                    │
│  + Symbolic links support                                           │
│                                                                     │
│  Recommendations:                                                   │
│  - Text files: Files.readString/writeString                         │
│  - Large files: Files.lines() with Stream                           │
│  - Binary: FileChannel with ByteBuffer                              │
│  - Network: NIO with Selectors                                      │
│  - Directory operations: Files + Path                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## На интервью

### Типичные вопросы

1. **InputStream vs Reader?**
   - InputStream: bytes (binary)
   - Reader: characters (text with encoding)
   - InputStreamReader: bridge between them

2. **Why BufferedReader/Writer?**
   - Reduces system calls
   - Reads/writes in chunks
   - readLine() convenience
   - Always use for performance

3. **NIO vs IO?**
   - IO: blocking, stream-based
   - NIO: non-blocking possible, buffer-based, channels
   - NIO for high-concurrency servers

4. **What is Selector?**
   - Multiplexes channels
   - One thread handles many connections
   - Key for scalable servers

5. **Path vs File?**
   - Path: modern, immutable, symbolic links
   - File: legacy, mutable
   - Use Path with Files class

6. **Memory-mapped files?**
   - File mapped to memory region
   - Fast random access
   - OS handles paging
   - Good for large files

---

[← Назад к списку тем](README.md)
