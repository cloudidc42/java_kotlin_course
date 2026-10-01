# Part 14: File I/O
## ขั้นตอนที่ 891-960: การอ่านเขียนไฟล์และ NIO.2

---

## 14.1 Java NIO.2 (java.nio.file)

```java
import java.io.*;
import java.nio.file.*;
import java.nio.charset.StandardCharsets;
import java.util.List;

public class NIOBasics {
    
    public static void main(String[] args) throws Exception {
        
        // ====== Path API ======
        Path current = Path.of(".");
        Path home = Path.of(System.getProperty("user.home"));
        Path file = Path.of("src", "main", "java", "Hello.java");
        Path absolute = file.toAbsolutePath();
        
        System.out.println("Current: " + current.toAbsolutePath());
        System.out.println("Home: " + home);
        System.out.println("File: " + file);
        System.out.println("Absolute: " + absolute);
        System.out.println("Parent: " + absolute.getParent());
        System.out.println("Filename: " + absolute.getFileName());
        System.out.println("Extension: " + getExtension(absolute));
        
        // Path manipulation
        Path resolved = home.resolve("Desktop/notes.txt");
        System.out.println("Resolved: " + resolved);
        Path relative = home.relativize(resolved);
        System.out.println("Relative: " + relative);
        
        // ====== Writing files ======
        Path testDir = Path.of("test_io");
        Files.createDirectories(testDir);  // create if not exists
        
        Path textFile = testDir.resolve("hello.txt");
        Files.writeString(textFile, "Hello, Java NIO!\nLine 2\nLine 3\n");
        System.out.println("\nWrote: " + textFile);
        
        // Append
        Files.writeString(textFile, "Appended line\n", StandardOpenOption.APPEND);
        
        // Write lines
        Path linesFile = testDir.resolve("lines.txt");
        Files.write(linesFile, List.of("Alpha", "Beta", "Gamma", "Delta"));
        
        // ====== Reading files ======
        System.out.println("\n=== Read File ===");
        String content = Files.readString(textFile);
        System.out.print(content);
        
        System.out.println("\n=== Read Lines ===");
        List<String> lines = Files.readAllLines(linesFile);
        for (int i = 0; i < lines.size(); i++) {
            System.out.printf("%d: %s%n", i + 1, lines.get(i));
        }
        
        // ====== File operations ======
        Path copyDest = testDir.resolve("hello_copy.txt");
        Files.copy(textFile, copyDest, StandardCopyOption.REPLACE_EXISTING);
        
        Path moveDest = testDir.resolve("hello_moved.txt");
        Files.move(copyDest, moveDest, StandardCopyOption.REPLACE_EXISTING);
        
        // Check file info
        System.out.println("\n=== File Info ===");
        System.out.println("Exists: " + Files.exists(textFile));
        System.out.println("Size: " + Files.size(textFile) + " bytes");
        System.out.println("Is file: " + Files.isRegularFile(textFile));
        System.out.println("Is dir: " + Files.isDirectory(testDir));
        System.out.println("Readable: " + Files.isReadable(textFile));
        System.out.println("Last modified: " + Files.getLastModifiedTime(textFile));
        
        // ====== Delete ======
        Files.deleteIfExists(moveDest);
        
        // Cleanup
        Files.deleteIfExists(linesFile);
        Files.deleteIfExists(textFile);
        Files.deleteIfExists(testDir);
    }
    
    static String getExtension(Path path) {
        String name = path.getFileName().toString();
        int dot = name.lastIndexOf('.');
        return dot > 0 ? name.substring(dot + 1) : "";
    }
}
```

---

## 14.2 BufferedReader/Writer และ Stream

```java
import java.io.*;
import java.nio.file.*;
import java.util.stream.Stream;

public class BufferedIODemo {
    
    // Write with BufferedWriter
    static void writeLargeFile(Path path, int lines) throws IOException {
        try (BufferedWriter writer = Files.newBufferedWriter(path)) {
            for (int i = 1; i <= lines; i++) {
                writer.write(String.format("Line %5d: Data content here%n", i));
            }
        }
        System.out.println("Wrote " + lines + " lines to " + path);
    }
    
    // Read with BufferedReader
    static long countWords(Path path) throws IOException {
        long wordCount = 0;
        try (BufferedReader reader = Files.newBufferedReader(path)) {
            String line;
            while ((line = reader.readLine()) != null) {
                wordCount += line.split("\\s+").length;
            }
        }
        return wordCount;
    }
    
    // Read with Stream<String> (lazy)
    static void processLargeFile(Path path) throws IOException {
        try (Stream<String> lines = Files.lines(path)) {
            lines.filter(l -> l.contains("100"))
                 .map(String::trim)
                 .limit(5)
                 .forEach(System.out::println);
        }
    }
    
    // CSV writing
    static void writeCsv(Path path, String[][] data) throws IOException {
        try (PrintWriter pw = new PrintWriter(Files.newBufferedWriter(path))) {
            for (String[] row : data) {
                pw.println(String.join(",", row));
            }
        }
    }
    
    // CSV reading
    static void readCsv(Path path) throws IOException {
        try (BufferedReader reader = Files.newBufferedReader(path)) {
            String header = reader.readLine();
            if (header == null) return;
            
            String[] cols = header.split(",");
            System.out.printf("%-15s %-20s %-8s %-8s%n", cols[0], cols[1], cols[2], cols[3]);
            System.out.println("-".repeat(55));
            
            String line;
            while ((line = reader.readLine()) != null) {
                String[] fields = line.split(",");
                if (fields.length >= 4) {
                    System.out.printf("%-15s %-20s %-8s %-8s%n",
                        fields[0], fields[1], fields[2], fields[3]);
                }
            }
        }
    }
    
    public static void main(String[] args) throws Exception {
        
        Path tmpDir = Path.of("tmp_io");
        Files.createDirectories(tmpDir);
        
        // Large file
        Path largeFile = tmpDir.resolve("large.txt");
        writeLargeFile(largeFile, 1000);
        System.out.println("Word count: " + countWords(largeFile));
        
        System.out.println("Lines containing '100':");
        processLargeFile(largeFile);
        
        // CSV
        Path csvFile = tmpDir.resolve("students.csv");
        String[][] data = {
            {"ID", "Name", "GPA", "Major"},
            {"S001", "Alice Smith", "3.80", "CS"},
            {"S002", "Bob Jones", "3.20", "Math"},
            {"S003", "Charlie Brown", "3.90", "CS"},
        };
        writeCsv(csvFile, data);
        
        System.out.println("\nCSV Content:");
        readCsv(csvFile);
        
        // Cleanup
        Files.deleteIfExists(largeFile);
        Files.deleteIfExists(csvFile);
        Files.deleteIfExists(tmpDir);
    }
}
```

---

## 14.3 Directory Walking

```java
import java.io.*;
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.util.*;
import java.util.stream.*;

public class DirectoryWalking {
    
    // Walk directory tree
    static void walkDirectory(Path dir) throws IOException {
        System.out.println("Directory tree: " + dir);
        
        Files.walk(dir)
             .forEach(path -> {
                 int depth = dir.relativize(path).getNameCount();
                 String indent = "  ".repeat(Files.isDirectory(path) ? depth - 1 : depth);
                 String icon = Files.isDirectory(path) ? "📁" : "📄";
                 System.out.println(indent + icon + " " + path.getFileName());
             });
    }
    
    // Find files by extension
    static List<Path> findByExtension(Path dir, String ext) throws IOException {
        try (Stream<Path> paths = Files.walk(dir)) {
            return paths.filter(Files::isRegularFile)
                        .filter(p -> p.toString().endsWith("." + ext))
                        .sorted()
                        .collect(Collectors.toList());
        }
    }
    
    // Calculate directory size
    static long directorySize(Path dir) throws IOException {
        try (Stream<Path> paths = Files.walk(dir)) {
            return paths.filter(Files::isRegularFile)
                        .mapToLong(p -> {
                            try { return Files.size(p); }
                            catch (IOException e) { return 0; }
                        })
                        .sum();
        }
    }
    
    // File visitor pattern
    static void visitWithCustomAction(Path dir) throws IOException {
        Files.walkFileTree(dir, new SimpleFileVisitor<>() {
            int depth = 0;
            
            @Override
            public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) {
                System.out.println("  ".repeat(depth) + "📁 " + dir.getFileName());
                depth++;
                return FileVisitResult.CONTINUE;
            }
            
            @Override
            public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
                System.out.printf("%s📄 %-20s %,d bytes%n",
                    "  ".repeat(depth), file.getFileName(), attrs.size());
                return FileVisitResult.CONTINUE;
            }
            
            @Override
            public FileVisitResult postVisitDirectory(Path dir, IOException exc) {
                depth--;
                return FileVisitResult.CONTINUE;
            }
        });
    }
    
    // Watch directory for changes
    static void watchDirectory(Path dir, int seconds) throws Exception {
        WatchService watcher = FileSystems.getDefault().newWatchService();
        dir.register(watcher,
            StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_DELETE,
            StandardWatchEventKinds.ENTRY_MODIFY);
        
        System.out.println("Watching: " + dir + " for " + seconds + "s");
        long end = System.currentTimeMillis() + seconds * 1000L;
        
        while (System.currentTimeMillis() < end) {
            WatchKey key = watcher.poll(100, java.util.concurrent.TimeUnit.MILLISECONDS);
            if (key == null) continue;
            
            for (WatchEvent<?> event : key.pollEvents()) {
                System.out.printf("[%s] %s%n", event.kind().name(), event.context());
            }
            key.reset();
        }
    }
    
    static void createSampleStructure(Path base) throws IOException {
        Files.createDirectories(base.resolve("src/main/java"));
        Files.createDirectories(base.resolve("src/test/java"));
        Files.createDirectories(base.resolve("resources"));
        
        Files.writeString(base.resolve("src/main/java/Main.java"), "class Main {}");
        Files.writeString(base.resolve("src/main/java/App.java"), "class App {}");
        Files.writeString(base.resolve("src/test/java/MainTest.java"), "class MainTest {}");
        Files.writeString(base.resolve("resources/config.properties"), "version=1.0");
        Files.writeString(base.resolve("resources/messages.txt"), "Hello World");
        Files.writeString(base.resolve("pom.xml"), "<project/>");
    }
    
    static void deleteDirectory(Path dir) throws IOException {
        if (!Files.exists(dir)) return;
        Files.walk(dir)
             .sorted(Comparator.reverseOrder())
             .forEach(p -> { try { Files.delete(p); } catch (IOException ignored) {} });
    }
    
    public static void main(String[] args) throws Exception {
        
        Path sampleDir = Path.of("sample_project");
        createSampleStructure(sampleDir);
        
        System.out.println("=== Directory Walk ===");
        walkDirectory(sampleDir);
        
        System.out.println("\n=== Java Files ===");
        findByExtension(sampleDir, "java")
            .forEach(p -> System.out.println("  " + p));
        
        System.out.println("\n=== File Visitor ===");
        visitWithCustomAction(sampleDir);
        
        System.out.printf("\nDirectory size: %,d bytes%n", directorySize(sampleDir));
        
        deleteDirectory(sampleDir);
    }
}
```

---

## 14.4 JSON-like File Formats

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.stream.*;

// Simple properties file handler
public class ConfigManager {
    private final Path configFile;
    private final Properties props = new Properties();
    
    public ConfigManager(String filename) {
        this.configFile = Path.of(filename);
        load();
    }
    
    private void load() {
        if (Files.exists(configFile)) {
            try (InputStream in = Files.newInputStream(configFile)) {
                props.load(in);
            } catch (IOException e) {
                System.err.println("Cannot load config: " + e.getMessage());
            }
        }
    }
    
    public void save() {
        try (OutputStream out = Files.newOutputStream(configFile)) {
            props.store(out, "App Configuration");
        } catch (IOException e) {
            System.err.println("Cannot save config: " + e.getMessage());
        }
    }
    
    public String get(String key, String defaultValue) {
        return props.getProperty(key, defaultValue);
    }
    
    public int getInt(String key, int defaultValue) {
        try { return Integer.parseInt(props.getProperty(key)); }
        catch (Exception e) { return defaultValue; }
    }
    
    public boolean getBoolean(String key, boolean defaultValue) {
        String val = props.getProperty(key);
        if (val == null) return defaultValue;
        return "true".equalsIgnoreCase(val) || "1".equals(val) || "yes".equalsIgnoreCase(val);
    }
    
    public void set(String key, Object value) {
        props.setProperty(key, String.valueOf(value));
    }
    
    public void printAll() {
        System.out.println("=== Configuration ===");
        props.stringPropertyNames().stream()
             .sorted()
             .forEach(k -> System.out.printf("  %-20s = %s%n", k, props.getProperty(k)));
    }
    
    // Simple CSV data class
    static class CsvTable {
        private final List<String> headers;
        private final List<List<String>> rows = new ArrayList<>();
        
        public CsvTable(List<String> headers) {
            this.headers = new ArrayList<>(headers);
        }
        
        public void addRow(String... values) {
            rows.add(Arrays.asList(values));
        }
        
        public void writeTo(Path path) throws IOException {
            try (PrintWriter pw = new PrintWriter(Files.newBufferedWriter(path))) {
                pw.println(String.join(",", headers));
                rows.forEach(row -> pw.println(String.join(",", row)));
            }
        }
        
        public static CsvTable readFrom(Path path) throws IOException {
            List<String> lines = Files.readAllLines(path);
            if (lines.isEmpty()) return new CsvTable(List.of());
            
            List<String> headers = Arrays.asList(lines.get(0).split(","));
            CsvTable table = new CsvTable(headers);
            
            for (int i = 1; i < lines.size(); i++) {
                table.rows.add(Arrays.asList(lines.get(i).split(",")));
            }
            return table;
        }
        
        public void print() {
            int[] widths = new int[headers.size()];
            for (int i = 0; i < headers.size(); i++) {
                widths[i] = headers.get(i).length();
                for (List<String> row : rows) {
                    if (i < row.size()) widths[i] = Math.max(widths[i], row.get(i).length());
                }
            }
            
            String format = Arrays.stream(widths)
                .mapToObj(w -> "%-" + (w + 2) + "s")
                .collect(Collectors.joining("|"));
            
            System.out.println(String.format(format, headers.toArray()));
            System.out.println("-".repeat(Arrays.stream(widths).sum() + widths.length * 3));
            rows.forEach(row -> System.out.println(String.format(format, row.toArray())));
        }
    }
    
    public static void main(String[] args) throws Exception {
        
        // Config file
        ConfigManager config = new ConfigManager("app.properties");
        config.set("app.name", "My Java App");
        config.set("app.version", "1.0.0");
        config.set("server.port", 8080);
        config.set("server.host", "localhost");
        config.set("debug.enabled", true);
        config.set("max.connections", 100);
        config.save();
        
        // Read back
        ConfigManager loaded = new ConfigManager("app.properties");
        loaded.printAll();
        System.out.println("\nPort: " + loaded.getInt("server.port", 3000));
        System.out.println("Debug: " + loaded.getBoolean("debug.enabled", false));
        
        // CSV Table
        CsvTable table = new CsvTable(List.of("ID", "Name", "Score", "Grade"));
        table.addRow("1", "Alice", "95", "A");
        table.addRow("2", "Bob", "82", "B");
        table.addRow("3", "Charlie", "78", "C");
        table.addRow("4", "Diana", "91", "A");
        
        Path csvPath = Path.of("grades.csv");
        table.writeTo(csvPath);
        
        System.out.println("\nCSV Table:");
        CsvTable.readFrom(csvPath).print();
        
        // Cleanup
        Files.deleteIfExists(Path.of("app.properties"));
        Files.deleteIfExists(csvPath);
    }
}
```

---

## 14.5 Serialization

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;

// Serializable class
class UserProfile implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private String username;
    private String email;
    private int age;
    private List<String> roles;
    private transient String password; // not serialized
    
    public UserProfile(String username, String email, int age) {
        this.username = username;
        this.email = email;
        this.age = age;
        this.roles = new ArrayList<>();
    }
    
    public void addRole(String role) { roles.add(role); }
    public void setPassword(String password) { this.password = password; }
    
    @Override
    public String toString() {
        return String.format("User{%s, %s, age=%d, roles=%s, password=%s}",
            username, email, age, roles, password);
    }
}

class ObjectSerializer {
    
    public static void serialize(Object obj, Path path) throws IOException {
        Files.createDirectories(path.getParent() != null ? path.getParent() : Path.of("."));
        try (ObjectOutputStream oos = new ObjectOutputStream(Files.newOutputStream(path))) {
            oos.writeObject(obj);
        }
    }
    
    @SuppressWarnings("unchecked")
    public static <T> T deserialize(Path path, Class<T> type) throws IOException, ClassNotFoundException {
        try (ObjectInputStream ois = new ObjectInputStream(Files.newInputStream(path))) {
            Object obj = ois.readObject();
            return type.cast(obj);
        }
    }
    
    public static void main(String[] args) throws Exception {
        
        UserProfile user = new UserProfile("alice", "alice@example.com", 28);
        user.addRole("ADMIN");
        user.addRole("USER");
        user.setPassword("secret123");
        
        System.out.println("Before serialize: " + user);
        
        Path dataFile = Path.of("user.dat");
        serialize(user, dataFile);
        System.out.println("Serialized to: " + dataFile + " (" + Files.size(dataFile) + " bytes)");
        
        UserProfile loaded = deserialize(dataFile, UserProfile.class);
        System.out.println("After deserialize: " + loaded);
        // password will be null (transient)
        
        Files.deleteIfExists(dataFile);
    }
}
```

---

## 14.6 Full Program: Log File Analyzer

```java
import java.io.*;
import java.nio.file.*;
import java.time.*;
import java.time.format.*;
import java.util.*;
import java.util.stream.*;
import java.util.regex.*;

public class LogAnalyzer {
    
    enum LogLevel { DEBUG, INFO, WARN, ERROR, FATAL }
    
    record LogEntry(LocalDateTime timestamp, LogLevel level, String source, String message) {}
    
    // Parse log line: "2024-01-15 10:30:00 [ERROR] com.example.App - Connection failed"
    static final Pattern LOG_PATTERN = Pattern.compile(
        "(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2}) \\[(\\w+)\\] ([\\w.]+) - (.+)"
    );
    static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    static Optional<LogEntry> parseLine(String line) {
        Matcher m = LOG_PATTERN.matcher(line);
        if (!m.matches()) return Optional.empty();
        try {
            LocalDateTime ts = LocalDateTime.parse(m.group(1), FMT);
            LogLevel level = LogLevel.valueOf(m.group(2));
            return Optional.of(new LogEntry(ts, level, m.group(3), m.group(4)));
        } catch (Exception e) {
            return Optional.empty();
        }
    }
    
    static void generateSampleLog(Path path) throws IOException {
        String[] sources = {"com.example.App", "com.example.DB", "com.example.API"};
        String[] messages = {
            "Connection established", "Query executed", "Request processed",
            "Connection failed", "Timeout occurred", "Invalid input",
            "NullPointerException", "Database error", "Authentication failed"
        };
        LogLevel[] levels = {LogLevel.DEBUG, LogLevel.INFO, LogLevel.WARN, LogLevel.ERROR, LogLevel.FATAL};
        int[] levelWeights = {10, 50, 20, 15, 5};
        
        Random rng = new Random(42);
        LocalDateTime start = LocalDateTime.of(2024, 1, 15, 8, 0, 0);
        
        try (PrintWriter pw = new PrintWriter(Files.newBufferedWriter(path))) {
            for (int i = 0; i < 200; i++) {
                start = start.plusSeconds(rng.nextInt(60));
                LogLevel level = weightedRandom(levels, levelWeights, rng);
                String source = sources[rng.nextInt(sources.length)];
                String msg = messages[rng.nextInt(messages.length)];
                pw.printf("%s [%s] %s - %s%n", start.format(FMT), level, source, msg);
            }
        }
    }
    
    static <T> T weightedRandom(T[] items, int[] weights, Random rng) {
        int total = Arrays.stream(weights).sum();
        int r = rng.nextInt(total);
        for (int i = 0; i < items.length; i++) {
            r -= weights[i];
            if (r < 0) return items[i];
        }
        return items[items.length - 1];
    }
    
    static void analyze(Path logFile) throws IOException {
        List<LogEntry> entries;
        try (Stream<String> lines = Files.lines(logFile)) {
            entries = lines.map(LogAnalyzer::parseLine)
                          .flatMap(Optional::stream)
                          .collect(Collectors.toList());
        }
        
        System.out.println("=== Log Analysis Report ===");
        System.out.println("Total entries: " + entries.size());
        
        // By level
        System.out.println("\nBy Level:");
        entries.stream()
               .collect(Collectors.groupingBy(LogEntry::level, Collectors.counting()))
               .entrySet().stream()
               .sorted(Map.Entry.comparingByKey())
               .forEach(e -> System.out.printf("  %-6s: %3d%n", e.getKey(), e.getValue()));
        
        // By source
        System.out.println("\nBy Source:");
        entries.stream()
               .collect(Collectors.groupingBy(LogEntry::source, Collectors.counting()))
               .entrySet().stream()
               .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
               .forEach(e -> System.out.printf("  %-30s: %3d%n", e.getKey(), e.getValue()));
        
        // Errors only
        System.out.println("\nERROR/FATAL messages:");
        entries.stream()
               .filter(e -> e.level() == LogLevel.ERROR || e.level() == LogLevel.FATAL)
               .limit(5)
               .forEach(e -> System.out.printf("  [%s] %s%n", e.timestamp(), e.message()));
        
        // Hourly distribution
        System.out.println("\nHourly distribution:");
        entries.stream()
               .collect(Collectors.groupingBy(
                   e -> e.timestamp().getHour(),
                   Collectors.counting()
               ))
               .entrySet().stream()
               .sorted(Map.Entry.comparingByKey())
               .forEach(e -> {
                   String bar = "█".repeat((int)(e.getValue() / 2));
                   System.out.printf("  %02d:00  %s %d%n", e.getKey(), bar, e.getValue());
               });
    }
    
    public static void main(String[] args) throws Exception {
        Path logFile = Path.of("app.log");
        generateSampleLog(logFile);
        analyze(logFile);
        Files.deleteIfExists(logFile);
    }
}
```

---

## สรุป Part 14

| หัวข้อ | API หลัก |
|--------|---------|
| Path API | `Path.of()`, `resolve()`, `relativize()` |
| Files API | `readString()`, `writeString()`, `copy()`, `move()` |
| BufferedReader/Writer | `Files.newBufferedReader/Writer()` |
| Stream<String> | `Files.lines()` - lazy reading |
| Directory Walking | `Files.walk()`, `walkFileTree()` |
| WatchService | ตรวจสอบการเปลี่ยนแปลงไฟล์ |
| Serialization | `ObjectOutputStream/InputStream` |

➡️ [Part 15: Multithreading](./Part-15-Multithreading.md)
