# Part 05: Arrays และ Multidimensional Arrays
## ขั้นตอนที่ 261-330: การจัดการข้อมูลแบบอาเรย์

---

## 5.1 Arrays พื้นฐาน

```java
import java.util.Arrays;

public class ArrayBasics {
    
    public static void main(String[] args) {
        
        // ====== การประกาศ Array ======
        
        // วิธีที่ 1: ประกาศแล้วกำหนดค่า
        int[] numbers = new int[5];  // {0,0,0,0,0} ค่าเริ่มต้น
        numbers[0] = 10;
        numbers[1] = 20;
        numbers[2] = 30;
        numbers[3] = 40;
        numbers[4] = 50;
        
        // วิธีที่ 2: ประกาศพร้อมกำหนดค่า
        int[] primes = {2, 3, 5, 7, 11, 13, 17, 19, 23, 29};
        
        // วิธีที่ 3: new array literal
        String[] fruits = new String[]{"Apple", "Banana", "Cherry"};
        
        // ====== การเข้าถึงข้อมูล ======
        System.out.println("numbers[0] = " + numbers[0]);
        System.out.println("primes[4] = " + primes[4]);  // 11
        System.out.println("Fruit 1: " + fruits[1]);      // Banana
        
        // ====== Length ======
        System.out.println("Length of numbers: " + numbers.length);   // 5
        System.out.println("Length of primes: " + primes.length);     // 10
        
        // ====== Looping ======
        System.out.print("Numbers: ");
        for (int i = 0; i < numbers.length; i++) {
            System.out.print(numbers[i] + " ");
        }
        System.out.println();
        
        // Enhanced for-each
        System.out.print("Primes: ");
        for (int prime : primes) {
            System.out.print(prime + " ");
        }
        System.out.println();
        
        // ====== Arrays utility methods ======
        int[] arr = {5, 2, 8, 1, 9, 3, 7, 4, 6};
        
        System.out.println("Original: " + Arrays.toString(arr));
        
        Arrays.sort(arr);
        System.out.println("Sorted: " + Arrays.toString(arr));
        
        int index = Arrays.binarySearch(arr, 7);
        System.out.println("Index of 7: " + index);
        
        int[] copy = Arrays.copyOf(arr, arr.length);
        System.out.println("Copy: " + Arrays.toString(copy));
        
        int[] partial = Arrays.copyOfRange(arr, 2, 6);
        System.out.println("Partial copy [2,6): " + Arrays.toString(partial));
        
        Arrays.fill(copy, 0);
        System.out.println("Filled: " + Arrays.toString(copy));
        
        boolean equal = Arrays.equals(arr, copy);
        System.out.println("Arrays equal: " + equal);
    }
}
```

---

## 5.2 Array Operations

```java
import java.util.Arrays;

public class ArrayOperations {
    
    public static void main(String[] args) {
        
        int[] data = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5};
        
        // ====== หาค่าสูงสุด/ต่ำสุด ======
        int max = data[0];
        int min = data[0];
        
        for (int num : data) {
            if (num > max) max = num;
            if (num < min) min = num;
        }
        System.out.println("Max: " + max + ", Min: " + min);
        
        // ====== คำนวณค่าเฉลี่ย ======
        double sum = 0;
        for (int num : data) {
            sum += num;
        }
        double average = sum / data.length;
        System.out.printf("Average: %.2f%n", average);
        
        // ====== นับความถี่ ======
        System.out.println("\n=== ความถี่ของตัวเลข ===");
        int[] frequency = new int[10]; // สำหรับเลข 0-9
        for (int num : data) {
            frequency[num]++;
        }
        for (int i = 0; i < frequency.length; i++) {
            if (frequency[i] > 0) {
                System.out.println(i + " พบ " + frequency[i] + " ครั้ง");
            }
        }
        
        // ====== Reverse Array ======
        int[] original = {1, 2, 3, 4, 5};
        int[] reversed = new int[original.length];
        
        for (int i = 0; i < original.length; i++) {
            reversed[i] = original[original.length - 1 - i];
        }
        System.out.println("\nOriginal: " + Arrays.toString(original));
        System.out.println("Reversed: " + Arrays.toString(reversed));
        
        // In-place reverse
        int[] arr = {1, 2, 3, 4, 5, 6};
        for (int left = 0, right = arr.length - 1; left < right; left++, right--) {
            int temp = arr[left];
            arr[left] = arr[right];
            arr[right] = temp;
        }
        System.out.println("In-place reversed: " + Arrays.toString(arr));
        
        // ====== Array Rotation ======
        int[] rotate = {1, 2, 3, 4, 5};
        int k = 2; // หมุนซ้าย 2 ตำแหน่ง
        
        rotateLeft(rotate, k);
        System.out.println("Rotated left by " + k + ": " + Arrays.toString(rotate));
    }
    
    static void rotateLeft(int[] arr, int k) {
        k = k % arr.length;
        reverse(arr, 0, k - 1);
        reverse(arr, k, arr.length - 1);
        reverse(arr, 0, arr.length - 1);
    }
    
    static void reverse(int[] arr, int start, int end) {
        while (start < end) {
            int temp = arr[start];
            arr[start++] = arr[end];
            arr[end--] = temp;
        }
    }
}
```

---

## 5.3 Sorting Algorithms

```java
import java.util.Arrays;

public class SortingAlgorithms {
    
    public static void main(String[] args) {
        
        // ====== Bubble Sort ======
        int[] arr1 = {64, 34, 25, 12, 22, 11, 90};
        System.out.println("Before: " + Arrays.toString(arr1));
        bubbleSort(arr1);
        System.out.println("Bubble Sort: " + Arrays.toString(arr1));
        
        // ====== Selection Sort ======
        int[] arr2 = {64, 25, 12, 22, 11};
        selectionSort(arr2);
        System.out.println("Selection Sort: " + Arrays.toString(arr2));
        
        // ====== Insertion Sort ======
        int[] arr3 = {12, 11, 13, 5, 6};
        insertionSort(arr3);
        System.out.println("Insertion Sort: " + Arrays.toString(arr3));
        
        // ====== Merge Sort ======
        int[] arr4 = {38, 27, 43, 3, 9, 82, 10};
        mergeSort(arr4, 0, arr4.length - 1);
        System.out.println("Merge Sort: " + Arrays.toString(arr4));
        
        // ====== Quick Sort ======
        int[] arr5 = {10, 7, 8, 9, 1, 5};
        quickSort(arr5, 0, arr5.length - 1);
        System.out.println("Quick Sort: " + Arrays.toString(arr5));
        
        // ====== Performance comparison ======
        System.out.println("\n=== Performance Test ===");
        int size = 10000;
        int[] large = generateRandom(size);
        
        // Java built-in (TimSort - fastest)
        int[] copy = Arrays.copyOf(large, large.length);
        long start = System.nanoTime();
        Arrays.sort(copy);
        long end = System.nanoTime();
        System.out.printf("Arrays.sort: %.2f ms%n", (end - start) / 1e6);
    }
    
    // O(n²) - ช้าสำหรับข้อมูลมาก
    static void bubbleSort(int[] arr) {
        int n = arr.length;
        boolean swapped;
        for (int i = 0; i < n - 1; i++) {
            swapped = false;
            for (int j = 0; j < n - i - 1; j++) {
                if (arr[j] > arr[j + 1]) {
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                    swapped = true;
                }
            }
            if (!swapped) break; // Already sorted
        }
    }
    
    // O(n²)
    static void selectionSort(int[] arr) {
        int n = arr.length;
        for (int i = 0; i < n - 1; i++) {
            int minIdx = i;
            for (int j = i + 1; j < n; j++) {
                if (arr[j] < arr[minIdx]) {
                    minIdx = j;
                }
            }
            int temp = arr[minIdx];
            arr[minIdx] = arr[i];
            arr[i] = temp;
        }
    }
    
    // O(n²) best case O(n)
    static void insertionSort(int[] arr) {
        int n = arr.length;
        for (int i = 1; i < n; i++) {
            int key = arr[i];
            int j = i - 1;
            while (j >= 0 && arr[j] > key) {
                arr[j + 1] = arr[j];
                j--;
            }
            arr[j + 1] = key;
        }
    }
    
    // O(n log n)
    static void mergeSort(int[] arr, int l, int r) {
        if (l < r) {
            int m = (l + r) / 2;
            mergeSort(arr, l, m);
            mergeSort(arr, m + 1, r);
            merge(arr, l, m, r);
        }
    }
    
    static void merge(int[] arr, int l, int m, int r) {
        int n1 = m - l + 1;
        int n2 = r - m;
        int[] left = new int[n1];
        int[] right = new int[n2];
        
        System.arraycopy(arr, l, left, 0, n1);
        System.arraycopy(arr, m + 1, right, 0, n2);
        
        int i = 0, j = 0, k = l;
        while (i < n1 && j < n2) {
            if (left[i] <= right[j]) {
                arr[k++] = left[i++];
            } else {
                arr[k++] = right[j++];
            }
        }
        while (i < n1) arr[k++] = left[i++];
        while (j < n2) arr[k++] = right[j++];
    }
    
    // O(n log n) average
    static void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pi = partition(arr, low, high);
            quickSort(arr, low, pi - 1);
            quickSort(arr, pi + 1, high);
        }
    }
    
    static int partition(int[] arr, int low, int high) {
        int pivot = arr[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++) {
            if (arr[j] <= pivot) {
                i++;
                int temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
        
        int temp = arr[i + 1];
        arr[i + 1] = arr[high];
        arr[high] = temp;
        
        return i + 1;
    }
    
    static int[] generateRandom(int size) {
        int[] arr = new int[size];
        java.util.Random random = new java.util.Random();
        for (int i = 0; i < size; i++) {
            arr[i] = random.nextInt(10000);
        }
        return arr;
    }
}
```

---

## 5.4 Searching Algorithms

```java
import java.util.Arrays;

public class SearchingAlgorithms {
    
    public static void main(String[] args) {
        
        int[] sortedArr = {1, 3, 5, 7, 9, 11, 13, 15, 17, 19, 21};
        int target = 13;
        
        // ====== Linear Search O(n) ======
        int linearResult = linearSearch(sortedArr, target);
        System.out.println("Linear Search for " + target + ": index " + linearResult);
        
        // ====== Binary Search O(log n) ======
        int binaryResult = binarySearch(sortedArr, target);
        System.out.println("Binary Search for " + target + ": index " + binaryResult);
        
        // Java built-in
        int builtIn = Arrays.binarySearch(sortedArr, target);
        System.out.println("Arrays.binarySearch: index " + builtIn);
        
        // ====== Search in unsorted ======
        String[] names = {"Charlie", "Alice", "Bob", "David", "Eve"};
        
        System.out.println("\nLinear Search in String array:");
        int found = linearSearchString(names, "Bob");
        System.out.println("Found 'Bob' at index: " + found);
        
        // ====== Find first/last occurrence ======
        int[] nums = {1, 2, 2, 3, 3, 3, 4, 4, 5};
        
        System.out.println("\nFirst occurrence of 3: " + firstOccurrence(nums, 3));
        System.out.println("Last occurrence of 3: " + lastOccurrence(nums, 3));
        System.out.println("Count of 3: " + countOccurrence(nums, 3));
    }
    
    static int linearSearch(int[] arr, int target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i] == target) return i;
        }
        return -1;
    }
    
    static int binarySearch(int[] arr, int target) {
        int left = 0, right = arr.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            
            if (arr[mid] == target) return mid;
            else if (arr[mid] < target) left = mid + 1;
            else right = mid - 1;
        }
        return -1;
    }
    
    static int linearSearchString(String[] arr, String target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i].equals(target)) return i;
        }
        return -1;
    }
    
    static int firstOccurrence(int[] arr, int target) {
        int result = -1;
        int left = 0, right = arr.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (arr[mid] == target) {
                result = mid;
                right = mid - 1; // ค้นหาต่อทางซ้าย
            } else if (arr[mid] < target) left = mid + 1;
            else right = mid - 1;
        }
        return result;
    }
    
    static int lastOccurrence(int[] arr, int target) {
        int result = -1;
        int left = 0, right = arr.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (arr[mid] == target) {
                result = mid;
                left = mid + 1; // ค้นหาต่อทางขวา
            } else if (arr[mid] < target) left = mid + 1;
            else right = mid - 1;
        }
        return result;
    }
    
    static int countOccurrence(int[] arr, int target) {
        int first = firstOccurrence(arr, target);
        int last = lastOccurrence(arr, target);
        if (first == -1) return 0;
        return last - first + 1;
    }
}
```

---

## 5.5 Multidimensional Arrays

```java
import java.util.Arrays;

public class MultidimensionalArrays {
    
    public static void main(String[] args) {
        
        // ====== 2D Array ======
        int[][] matrix = new int[3][4];  // 3 rows, 4 columns
        
        // กำหนดค่า
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 4; j++) {
                matrix[i][j] = i * 4 + j + 1;
            }
        }
        
        // แสดงผล
        System.out.println("=== Matrix ===");
        for (int[] row : matrix) {
            for (int val : row) {
                System.out.printf("%4d", val);
            }
            System.out.println();
        }
        
        // ====== Matrix Initialization ======
        int[][] chess = {
            {1, 0, 1, 0, 1, 0, 1, 0},
            {0, 1, 0, 1, 0, 1, 0, 1},
            {1, 0, 1, 0, 1, 0, 1, 0},
            {0, 1, 0, 1, 0, 1, 0, 1}
        };
        
        // ====== Matrix Operations ======
        
        // Transpose
        int[][] original = {{1, 2, 3}, {4, 5, 6}};
        int rows = original.length;
        int cols = original[0].length;
        int[][] transposed = new int[cols][rows];
        
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {
                transposed[j][i] = original[i][j];
            }
        }
        
        System.out.println("\n=== Original ===");
        printMatrix(original);
        System.out.println("=== Transposed ===");
        printMatrix(transposed);
        
        // Matrix Multiplication
        int[][] a = {{1, 2}, {3, 4}};
        int[][] b = {{5, 6}, {7, 8}};
        int[][] result = multiplyMatrix(a, b);
        
        System.out.println("\n=== Matrix Multiplication ===");
        printMatrix(a);
        System.out.println("×");
        printMatrix(b);
        System.out.println("=");
        printMatrix(result);
        
        // ====== Jagged Array ======
        System.out.println("\n=== Jagged Array ===");
        int[][] jagged = new int[5][];
        for (int i = 0; i < jagged.length; i++) {
            jagged[i] = new int[i + 1];
            for (int j = 0; j <= i; j++) {
                jagged[i][j] = j + 1;
            }
        }
        
        for (int[] row : jagged) {
            System.out.println(Arrays.toString(row));
        }
        
        // ====== 3D Array ======
        System.out.println("\n=== 3D Array (2×3×4) ===");
        int[][][] cube = new int[2][3][4];
        
        for (int i = 0; i < 2; i++) {
            for (int j = 0; j < 3; j++) {
                for (int k = 0; k < 4; k++) {
                    cube[i][j][k] = i * 12 + j * 4 + k;
                }
            }
        }
        System.out.println("cube[1][2][3] = " + cube[1][2][3]);
    }
    
    static void printMatrix(int[][] matrix) {
        for (int[] row : matrix) {
            for (int val : row) {
                System.out.printf("%4d", val);
            }
            System.out.println();
        }
    }
    
    static int[][] multiplyMatrix(int[][] a, int[][] b) {
        int rowsA = a.length;
        int colsA = a[0].length;
        int colsB = b[0].length;
        
        int[][] result = new int[rowsA][colsB];
        
        for (int i = 0; i < rowsA; i++) {
            for (int j = 0; j < colsB; j++) {
                for (int k = 0; k < colsA; k++) {
                    result[i][j] += a[i][k] * b[k][j];
                }
            }
        }
        return result;
    }
}
```

---

## 5.6 Array กับ String

```java
import java.util.Arrays;

public class ArrayAndString {
    
    public static void main(String[] args) {
        
        // ====== String to char array ======
        String text = "Hello, Java!";
        char[] chars = text.toCharArray();
        
        System.out.println("String: " + text);
        System.out.print("Chars: ");
        for (char c : chars) {
            System.out.print("[" + c + "]");
        }
        System.out.println();
        
        // ====== char array to String ======
        char[] vowels = {'a', 'e', 'i', 'o', 'u'};
        String vowelStr = new String(vowels);
        System.out.println("Vowels: " + vowelStr);
        
        // ====== Split and Join ======
        String sentence = "Java is a programming language";
        String[] words = sentence.split(" ");
        
        System.out.println("\nWords:");
        for (String word : words) {
            System.out.println("  " + word);
        }
        
        String rejoined = String.join(" ", words);
        System.out.println("Rejoined: " + rejoined);
        
        // ====== Anagram Check ======
        String s1 = "listen";
        String s2 = "silent";
        System.out.println("\nAnagram check:");
        System.out.println(s1 + " & " + s2 + ": " + isAnagram(s1, s2));
        
        // ====== Count words frequency ======
        String text2 = "the quick brown fox jumps over the lazy dog the fox";
        String[] wordArr = text2.split("\\s+");
        
        System.out.println("\n=== Word Frequency ===");
        Arrays.sort(wordArr);
        
        String current = wordArr[0];
        int count = 1;
        
        for (int i = 1; i < wordArr.length; i++) {
            if (wordArr[i].equals(current)) {
                count++;
            } else {
                System.out.printf("%-10s: %d%n", current, count);
                current = wordArr[i];
                count = 1;
            }
        }
        System.out.printf("%-10s: %d%n", current, count);
        
        // ====== Character counting ======
        String hello = "Hello World";
        int[] charCount = new int[256]; // ASCII
        
        for (char c : hello.toCharArray()) {
            charCount[c]++;
        }
        
        System.out.println("\n=== Character Count ===");
        for (int i = 0; i < 256; i++) {
            if (charCount[i] > 0) {
                System.out.printf("'%c': %d%n", (char)i, charCount[i]);
            }
        }
    }
    
    static boolean isAnagram(String s1, String s2) {
        if (s1.length() != s2.length()) return false;
        
        char[] arr1 = s1.toLowerCase().toCharArray();
        char[] arr2 = s2.toLowerCase().toCharArray();
        
        Arrays.sort(arr1);
        Arrays.sort(arr2);
        
        return Arrays.equals(arr1, arr2);
    }
}
```

---

## 5.7 Dynamic Array (ArrayList)

```java
import java.util.*;

public class DynamicArray {
    
    public static void main(String[] args) {
        
        // ====== ArrayList vs Array ======
        // Array: fixed size, อย่างเดียวสำหรับ primitive
        int[] fixedArray = new int[5];
        
        // ArrayList: dynamic size, เก็บแค่ Object
        ArrayList<Integer> dynamicList = new ArrayList<>();
        
        // ====== ArrayList Operations ======
        ArrayList<String> names = new ArrayList<>();
        
        // เพิ่ม
        names.add("Alice");
        names.add("Bob");
        names.add("Charlie");
        names.add(1, "David"); // เพิ่มที่ index 1
        
        System.out.println("After add: " + names);
        
        // ลบ
        names.remove("Bob");           // ลบโดยค่า
        names.remove(0);               // ลบโดย index
        
        System.out.println("After remove: " + names);
        
        // เข้าถึง
        System.out.println("Index 0: " + names.get(0));
        System.out.println("Size: " + names.size());
        
        // แก้ไข
        names.set(0, "Alex");
        System.out.println("After set: " + names);
        
        // ค้นหา
        System.out.println("Contains Charlie: " + names.contains("Charlie"));
        System.out.println("Index of David: " + names.indexOf("David"));
        
        // ====== Collections utility ======
        ArrayList<Integer> numbers = new ArrayList<>(Arrays.asList(5, 3, 8, 1, 9, 2, 7));
        
        System.out.println("\nOriginal: " + numbers);
        Collections.sort(numbers);
        System.out.println("Sorted: " + numbers);
        Collections.reverse(numbers);
        System.out.println("Reversed: " + numbers);
        Collections.shuffle(numbers);
        System.out.println("Shuffled: " + numbers);
        System.out.println("Max: " + Collections.max(numbers));
        System.out.println("Min: " + Collections.min(numbers));
        
        // ====== Convert Array <-> ArrayList ======
        
        // Array to ArrayList
        String[] arr = {"Java", "Python", "Go"};
        List<String> listFromArr = new ArrayList<>(Arrays.asList(arr));
        listFromArr.add("Kotlin");
        System.out.println("\nList from array: " + listFromArr);
        
        // ArrayList to Array
        String[] arrFromList = listFromArr.toArray(new String[0]);
        System.out.println("Array from list: " + Arrays.toString(arrFromList));
        
        // ====== Iteration ======
        List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5);
        
        // forEach with lambda
        nums.forEach(n -> System.out.print(n * 2 + " "));
        System.out.println();
        
        // removeIf
        List<Integer> mutable = new ArrayList<>(nums);
        mutable.removeIf(n -> n % 2 == 0);
        System.out.println("After removeIf (even): " + mutable);
        
        // replaceAll
        mutable.replaceAll(n -> n * n);
        System.out.println("After replaceAll (square): " + mutable);
    }
}
```

---

## 5.8 โปรแกรมตัวอย่าง: Student Grade Management

```java
import java.util.Arrays;
import java.util.Scanner;

public class StudentGradeManagement {
    
    private static String[] names;
    private static double[] scores;
    private static int count = 0;
    private static final int MAX_STUDENTS = 50;
    
    public static void main(String[] args) {
        names = new String[MAX_STUDENTS];
        scores = new double[MAX_STUDENTS];
        
        Scanner scanner = new Scanner(System.in);
        boolean running = true;
        
        System.out.println("╔═════════════════════════════╗");
        System.out.println("║  ระบบจัดการเกรดนักศึกษา    ║");
        System.out.println("╚═════════════════════════════╝");
        
        while (running) {
            System.out.println("\n=== เมนู ===");
            System.out.println("1. เพิ่มนักศึกษา");
            System.out.println("2. แสดงรายชื่อทั้งหมด");
            System.out.println("3. แสดงสถิติ");
            System.out.println("4. ค้นหานักศึกษา");
            System.out.println("5. เรียงลำดับตามคะแนน");
            System.out.println("6. ออก");
            System.out.print("เลือก: ");
            
            int choice = scanner.nextInt();
            scanner.nextLine();
            
            switch (choice) {
                case 1: addStudent(scanner); break;
                case 2: displayAll(); break;
                case 3: showStatistics(); break;
                case 4: searchStudent(scanner); break;
                case 5: sortByScore(); break;
                case 6: running = false; break;
                default: System.out.println("กรุณาเลือก 1-6");
            }
        }
        
        System.out.println("ขอบคุณที่ใช้บริการ!");
        scanner.close();
    }
    
    private static void addStudent(Scanner scanner) {
        if (count >= MAX_STUDENTS) {
            System.out.println("เต็มแล้ว!");
            return;
        }
        
        System.out.print("ชื่อนักศึกษา: ");
        names[count] = scanner.nextLine();
        
        System.out.print("คะแนน (0-100): ");
        scores[count] = scanner.nextDouble();
        scanner.nextLine();
        
        if (scores[count] < 0 || scores[count] > 100) {
            System.out.println("คะแนนไม่ถูกต้อง!");
            return;
        }
        
        count++;
        System.out.println("เพิ่มสำเร็จ! รวม " + count + " คน");
    }
    
    private static void displayAll() {
        if (count == 0) {
            System.out.println("ยังไม่มีนักศึกษา");
            return;
        }
        
        System.out.println("\n╔═══════════════════════════════════════╗");
        System.out.println("║  #  ชื่อ              คะแนน   เกรด   ║");
        System.out.println("╠═══════════════════════════════════════╣");
        
        for (int i = 0; i < count; i++) {
            String grade = getGrade(scores[i]);
            System.out.printf("║ %2d  %-16s %6.1f  %3s    ║%n",
                i + 1, names[i], scores[i], grade);
        }
        System.out.println("╚═══════════════════════════════════════╝");
    }
    
    private static void showStatistics() {
        if (count == 0) {
            System.out.println("ยังไม่มีข้อมูล");
            return;
        }
        
        double sum = 0, maxScore = scores[0], minScore = scores[0];
        int passCount = 0;
        
        for (int i = 0; i < count; i++) {
            sum += scores[i];
            if (scores[i] > maxScore) maxScore = scores[i];
            if (scores[i] < minScore) minScore = scores[i];
            if (scores[i] >= 50) passCount++;
        }
        
        double avg = sum / count;
        
        System.out.println("\n=== สถิติ ===");
        System.out.printf("จำนวนนักศึกษา: %d คน%n", count);
        System.out.printf("คะแนนสูงสุด:   %.1f%n", maxScore);
        System.out.printf("คะแนนต่ำสุด:   %.1f%n", minScore);
        System.out.printf("คะแนนเฉลี่ย:  %.2f%n", avg);
        System.out.printf("ผ่าน: %d คน (%.1f%%)%n", passCount, (double)passCount/count*100);
        System.out.printf("ไม่ผ่าน: %d คน%n", count - passCount);
    }
    
    private static void searchStudent(Scanner scanner) {
        System.out.print("ค้นหาชื่อ: ");
        String search = scanner.nextLine().toLowerCase();
        
        boolean found = false;
        for (int i = 0; i < count; i++) {
            if (names[i].toLowerCase().contains(search)) {
                System.out.printf("พบ: %s - คะแนน: %.1f เกรด: %s%n",
                    names[i], scores[i], getGrade(scores[i]));
                found = true;
            }
        }
        
        if (!found) System.out.println("ไม่พบ: " + search);
    }
    
    private static void sortByScore() {
        // Bubble sort (ง่ายและชัดเจน)
        for (int i = 0; i < count - 1; i++) {
            for (int j = 0; j < count - i - 1; j++) {
                if (scores[j] < scores[j + 1]) {
                    double tempScore = scores[j];
                    scores[j] = scores[j + 1];
                    scores[j + 1] = tempScore;
                    
                    String tempName = names[j];
                    names[j] = names[j + 1];
                    names[j + 1] = tempName;
                }
            }
        }
        System.out.println("เรียงตามคะแนนสำเร็จ (มากไปน้อย)");
        displayAll();
    }
    
    private static String getGrade(double score) {
        if (score >= 80) return "A";
        if (score >= 70) return "B";
        if (score >= 60) return "C";
        if (score >= 50) return "D";
        return "F";
    }
}
```

---

## 5.9 แบบฝึกหัด Part 05

### แบบฝึกหัดที่ 1: ค้นหาตัวซ้ำ
```java
import java.util.Arrays;

public class FindDuplicates {
    public static void main(String[] args) {
        int[] arr = {1, 3, 4, 2, 2, 3, 1, 5, 6, 7, 7};
        
        System.out.print("ตัวซ้ำ: ");
        Arrays.sort(arr);
        for (int i = 0; i < arr.length - 1; i++) {
            if (arr[i] == arr[i + 1]) {
                System.out.print(arr[i] + " ");
                while (i + 1 < arr.length && arr[i] == arr[i + 1]) i++;
            }
        }
        System.out.println();
    }
}
```

### แบบฝึกหัดที่ 2: Spiral Matrix
```java
public class SpiralMatrix {
    public static void main(String[] args) {
        int n = 4;
        int[][] spiral = new int[n][n];
        int num = 1;
        int top = 0, bottom = n - 1, left = 0, right = n - 1;
        
        while (num <= n * n) {
            for (int i = left; i <= right && num <= n * n; i++) spiral[top][i] = num++;
            top++;
            for (int i = top; i <= bottom && num <= n * n; i++) spiral[i][right] = num++;
            right--;
            for (int i = right; i >= left && num <= n * n; i--) spiral[bottom][i] = num++;
            bottom--;
            for (int i = bottom; i >= top && num <= n * n; i--) spiral[i][left] = num++;
            left++;
        }
        
        System.out.println("=== Spiral Matrix " + n + "×" + n + " ===");
        for (int[] row : spiral) {
            for (int val : row) System.out.printf("%3d", val);
            System.out.println();
        }
    }
}
```

---

## สรุป Part 05

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Array Basics | การประกาศ, เข้าถึง, ความยาว |
| Array Operations | หาสูงสุด/ต่ำสุด, ค่าเฉลี่ย, ความถี่ |
| Sorting | Bubble, Selection, Insertion, Merge, Quick Sort |
| Searching | Linear Search, Binary Search |
| 2D Arrays | Matrix operations, transpose, multiplication |
| Dynamic Arrays | ArrayList, Collections |

---

## ขั้นตอนต่อไป

➡️ [Part 06: Methods และ Functions](./Part-06-Methods.md)
