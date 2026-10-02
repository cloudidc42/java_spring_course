# Part 006: Arrays & Collections Basics
## หลักสูตร Java & Spring Boot ฉบับสมบูรณ์

---

## สารบัญ
1. [Arrays พื้นฐาน](#arrays-พื้นฐาน)
2. [Multi-dimensional Arrays](#multi-dimensional-arrays)
3. [Arrays Class](#arrays-class)
4. [ArrayList](#arraylist)
5. [LinkedList](#linkedlist)
6. [Stack](#stack)
7. [Queue และ Deque](#queue-และ-deque)
8. [Collections Utility Methods](#collections-utility-methods)
9. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Arrays พื้นฐาน

### การประกาศและสร้าง Array

```java
import java.util.Arrays;

public class ArrayBasics {
    public static void main(String[] args) {
        // สร้าง array แบบต่างๆ
        
        // 1. ประกาศขนาดก่อน
        int[] numbers = new int[5];       // [0, 0, 0, 0, 0]
        String[] names = new String[3];   // [null, null, null]
        
        // 2. ประกาศพร้อมค่า
        int[] scores = {85, 92, 78, 90, 88};
        String[] fruits = {"apple", "banana", "cherry"};
        
        // 3. Anonymous array
        printArray(new int[]{1, 2, 3, 4, 5});
        
        // Accessing elements
        System.out.println("scores[0] = " + scores[0]);  // 85
        System.out.println("scores[4] = " + scores[4]);  // 88
        System.out.println("length = " + scores.length); // 5
        
        // ดัชนีสุดท้าย
        System.out.println("Last: " + scores[scores.length - 1]);
        
        // ArrayIndexOutOfBoundsException ถ้าเกิน index
        // System.out.println(scores[5]);  // Error!
        
        // Modify element
        scores[2] = 95;
        System.out.println("After modify: " + Arrays.toString(scores));
        
        // Iterate
        System.out.print("for loop: ");
        for (int i = 0; i < scores.length; i++) {
            System.out.print(scores[i] + " ");
        }
        System.out.println();
        
        System.out.print("for-each: ");
        for (int score : scores) {
            System.out.print(score + " ");
        }
        System.out.println();
        
        // Default values
        int[] intArr = new int[3];       // [0, 0, 0]
        double[] dblArr = new double[3]; // [0.0, 0.0, 0.0]
        boolean[] boolArr = new boolean[3]; // [false, false, false]
        String[] strArr = new String[3]; // [null, null, null]
        System.out.println("\nDefault int: " + Arrays.toString(intArr));
        System.out.println("Default String: " + Arrays.toString(strArr));
    }
    
    static void printArray(int[] arr) {
        System.out.println("Array: " + Arrays.toString(arr));
    }
}
```

### Array Operations

```java
import java.util.Arrays;

public class ArrayOperations {
    public static void main(String[] args) {
        int[] arr = {64, 34, 25, 12, 22, 11, 90};
        
        // Copy array
        int[] copy1 = arr.clone();
        int[] copy2 = Arrays.copyOf(arr, arr.length);
        int[] copy3 = Arrays.copyOfRange(arr, 2, 5);  // [25, 12, 22]
        
        System.out.println("Original: " + Arrays.toString(arr));
        System.out.println("Clone: " + Arrays.toString(copy1));
        System.out.println("CopyOfRange[2,5]: " + Arrays.toString(copy3));
        
        // Sort
        int[] toSort = arr.clone();
        Arrays.sort(toSort);
        System.out.println("\nSorted: " + Arrays.toString(toSort));
        
        // Sort descending (ต้องใช้ Integer[])
        Integer[] descArr = {5, 3, 8, 1, 9, 2, 7};
        Arrays.sort(descArr, (a, b) -> b - a);
        System.out.println("Descending: " + Arrays.toString(descArr));
        
        // Binary search (ต้อง sort ก่อน)
        int[] sorted = {1, 3, 5, 7, 9, 11, 13};
        int idx = Arrays.binarySearch(sorted, 7);
        System.out.println("\nBinary search 7: index " + idx);
        
        // Fill
        int[] filled = new int[5];
        Arrays.fill(filled, 42);
        System.out.println("Filled: " + Arrays.toString(filled));
        
        // Compare arrays
        int[] a = {1, 2, 3};
        int[] b = {1, 2, 3};
        int[] c = {1, 2, 4};
        System.out.println("\na equals b: " + Arrays.equals(a, b));  // true
        System.out.println("a equals c: " + Arrays.equals(a, c));   // false
        
        // Sum, min, max
        int[] data = {3, 1, 4, 1, 5, 9, 2, 6};
        int sum = 0;
        for (int n : data) sum += n;
        System.out.println("\nSum: " + sum);
        System.out.println("Min: " + Arrays.stream(data).min().getAsInt());
        System.out.println("Max: " + Arrays.stream(data).max().getAsInt());
    }
}
```

---

## Multi-dimensional Arrays

```java
import java.util.Arrays;

public class MultiDimensionalArrays {
    public static void main(String[] args) {
        // 2D array
        int[][] matrix = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };
        
        // Access: matrix[row][col]
        System.out.println("matrix[1][2] = " + matrix[1][2]);  // 6
        
        // Print 2D array
        System.out.println("\nMatrix:");
        for (int[] row : matrix) {
            System.out.println(Arrays.toString(row));
        }
        
        // หรือใช้ Arrays.deepToString
        System.out.println(Arrays.deepToString(matrix));
        
        // Jagged array (array of different sizes)
        int[][] jagged = new int[3][];
        jagged[0] = new int[]{1};
        jagged[1] = new int[]{2, 3};
        jagged[2] = new int[]{4, 5, 6};
        
        System.out.println("\nJagged:");
        for (int[] row : jagged) {
            System.out.println(Arrays.toString(row));
        }
        
        // Matrix addition
        int[][] A = {{1, 2}, {3, 4}};
        int[][] B = {{5, 6}, {7, 8}};
        int[][] C = new int[2][2];
        
        for (int i = 0; i < 2; i++) {
            for (int j = 0; j < 2; j++) {
                C[i][j] = A[i][j] + B[i][j];
            }
        }
        System.out.println("\nA + B = " + Arrays.deepToString(C));
        
        // Transpose matrix
        int rows = matrix.length, cols = matrix[0].length;
        int[][] transposed = new int[cols][rows];
        for (int i = 0; i < rows; i++) {
            for (int j = 0; j < cols; j++) {
                transposed[j][i] = matrix[i][j];
            }
        }
        System.out.println("\nTransposed: " + Arrays.deepToString(transposed));
        
        // 3D array
        int[][][] cube = new int[3][3][3];
        int val = 1;
        for (int i = 0; i < 3; i++)
            for (int j = 0; j < 3; j++)
                for (int k = 0; k < 3; k++)
                    cube[i][j][k] = val++;
        System.out.println("\n3D: cube[0][0] = " + Arrays.toString(cube[0][0]));
    }
}
```

---

## Arrays Class

```java
import java.util.Arrays;

public class ArraysClass {
    public static void main(String[] args) {
        // Arrays.toString - แสดง array เป็น String
        int[] arr = {1, 2, 3, 4, 5};
        System.out.println(Arrays.toString(arr));  // [1, 2, 3, 4, 5]
        
        // Arrays.sort
        int[] unsorted = {5, 3, 8, 1, 9, 2, 7};
        Arrays.sort(unsorted);
        System.out.println(Arrays.toString(unsorted));
        
        // Partial sort
        int[] partial = {5, 3, 8, 1, 9, 2, 7};
        Arrays.sort(partial, 2, 5);  // sort index 2 to 4
        System.out.println("Partial sort: " + Arrays.toString(partial));
        
        // Arrays.fill
        int[] filled = new int[5];
        Arrays.fill(filled, 7);
        System.out.println("Filled: " + Arrays.toString(filled));
        Arrays.fill(filled, 1, 4, 0);  // fill index 1-3 with 0
        System.out.println("Partial fill: " + Arrays.toString(filled));
        
        // Arrays.copyOf
        int[] original = {1, 2, 3, 4, 5};
        int[] shorter = Arrays.copyOf(original, 3);    // [1, 2, 3]
        int[] longer = Arrays.copyOf(original, 8);     // [1, 2, 3, 4, 5, 0, 0, 0]
        System.out.println("Shorter: " + Arrays.toString(shorter));
        System.out.println("Longer: " + Arrays.toString(longer));
        
        // Arrays.stream - แปลงเป็น Stream (จะเรียนใน Part 014)
        int sum = Arrays.stream(original).sum();
        double avg = Arrays.stream(original).average().orElse(0);
        System.out.println("Sum: " + sum + ", Avg: " + avg);
        
        // Convert between array and list
        String[] strArr = {"apple", "banana", "cherry"};
        java.util.List<String> list = Arrays.asList(strArr);
        System.out.println("List: " + list);
        
        // Note: Arrays.asList ให้ fixed-size list
        // list.add("date");  // UnsupportedOperationException!
        
        // ใช้ ArrayList ถ้าต้องการ add/remove
        java.util.ArrayList<String> mutableList = new java.util.ArrayList<>(Arrays.asList(strArr));
        mutableList.add("date");
        System.out.println("Mutable list: " + mutableList);
    }
}
```

---

## ArrayList

ArrayList เป็น dynamic array ที่ขยายขนาดได้อัตโนมัติ

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.Iterator;
import java.util.List;

public class ArrayListExample {
    public static void main(String[] args) {
        // สร้าง ArrayList
        ArrayList<String> fruits = new ArrayList<>();
        
        // เพิ่ม elements
        fruits.add("apple");
        fruits.add("banana");
        fruits.add("cherry");
        fruits.add(0, "avocado");   // เพิ่มที่ index 0
        System.out.println("Fruits: " + fruits);
        
        // Access
        System.out.println("Index 1: " + fruits.get(1));
        System.out.println("Size: " + fruits.size());
        System.out.println("Contains banana: " + fruits.contains("banana"));
        System.out.println("Index of cherry: " + fruits.indexOf("cherry"));
        
        // Update
        fruits.set(0, "apricot");
        System.out.println("After set: " + fruits);
        
        // Remove
        fruits.remove("banana");            // remove by value
        fruits.remove(0);                   // remove by index
        System.out.println("After remove: " + fruits);
        
        // Iterate ทุกวิธี
        System.out.println("\nfor-each:");
        for (String fruit : fruits) {
            System.out.println("  " + fruit);
        }
        
        System.out.println("\niterator:");
        Iterator<String> it = fruits.iterator();
        while (it.hasNext()) {
            System.out.println("  " + it.next());
        }
        
        System.out.println("\nforEach lambda:");
        fruits.forEach(f -> System.out.println("  " + f));
        
        // Sort
        ArrayList<Integer> numbers = new ArrayList<>(List.of(5, 3, 8, 1, 9, 2, 7));
        Collections.sort(numbers);
        System.out.println("\nSorted: " + numbers);
        
        Collections.sort(numbers, (a, b) -> b - a);  // descending
        System.out.println("Descending: " + numbers);
        
        // SubList
        List<Integer> sub = numbers.subList(1, 4);
        System.out.println("SubList[1,4]: " + sub);
        
        // Convert to array
        String[] arr = fruits.toArray(new String[0]);
        System.out.println("\nTo array: " + java.util.Arrays.toString(arr));
        
        // Clear
        fruits.clear();
        System.out.println("After clear: " + fruits + " isEmpty: " + fruits.isEmpty());
        
        // Nested ArrayList (2D)
        ArrayList<ArrayList<Integer>> matrix = new ArrayList<>();
        for (int i = 0; i < 3; i++) {
            ArrayList<Integer> row = new ArrayList<>();
            for (int j = 0; j < 3; j++) {
                row.add(i * 3 + j + 1);
            }
            matrix.add(row);
        }
        System.out.println("\nMatrix: " + matrix);
    }
}
```

### ArrayList vs Array

```java
import java.util.ArrayList;
import java.util.List;

public class ArrayVsArrayList {
    public static void main(String[] args) {
        // Array: ขนาดคงที่
        int[] arr = new int[5];
        // arr.length = 5 ตลอด
        
        // ArrayList: ขนาดยืดหยุ่น
        ArrayList<Integer> list = new ArrayList<>();
        for (int i = 1; i <= 10; i++) list.add(i);
        System.out.println("ArrayList size: " + list.size());
        
        // Performance differences:
        // Array:     เร็วกว่าในการ access และ iterate
        // ArrayList: เร็วกว่าในการ add/remove
        
        // ใช้ List interface (ดีกว่า)
        List<String> names = new ArrayList<>();
        names.add("Alice");
        names.add("Bob");
        names.add("Charlie");
        
        // สามารถเปลี่ยน implementation ได้ง่าย
        // List<String> names = new LinkedList<>();  // เปลี่ยนได้ทันที
        
        // List.of (Java 9+) - immutable
        List<String> immutable = List.of("a", "b", "c");
        // immutable.add("d");  // UnsupportedOperationException!
        
        // List.copyOf (Java 10+)
        List<String> copy = List.copyOf(names);
        System.out.println("Immutable: " + immutable);
        System.out.println("Copy: " + copy);
    }
}
```

---

## LinkedList

```java
import java.util.LinkedList;

public class LinkedListExample {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>();
        
        // เพิ่มที่หัวและท้าย
        list.add("B");
        list.addFirst("A");   // เพิ่มที่หัว
        list.addLast("C");    // เพิ่มที่ท้าย
        list.add(1, "AB");    // เพิ่มที่ index
        System.out.println("List: " + list);
        
        // Access
        System.out.println("First: " + list.getFirst());
        System.out.println("Last: " + list.getLast());
        System.out.println("Index 1: " + list.get(1));
        
        // Remove
        list.removeFirst();
        list.removeLast();
        System.out.println("After remove first/last: " + list);
        
        // LinkedList เป็นทั้ง List และ Deque
        LinkedList<Integer> deque = new LinkedList<>();
        deque.push(1);    // เพิ่มหัว (stack push)
        deque.push(2);
        deque.push(3);
        System.out.println("\nStack (push): " + deque);
        System.out.println("Pop: " + deque.pop());  // ดึงจากหัว
        System.out.println("Peek: " + deque.peek()); // ดูหัว ไม่ดึง
        
        // Queue operations
        LinkedList<String> queue = new LinkedList<>();
        queue.offer("First");   // เพิ่มท้าย
        queue.offer("Second");
        queue.offer("Third");
        System.out.println("\nQueue: " + queue);
        System.out.println("Poll: " + queue.poll());  // ดึงจากหัว
        System.out.println("Queue after poll: " + queue);
        
        // Performance: LinkedList ดีกว่าใน insert/delete
        // แต่ ArrayList ดีกว่าใน random access
    }
}
```

---

## Stack

```java
import java.util.Stack;
import java.util.ArrayDeque;
import java.util.Deque;

public class StackExample {
    public static void main(String[] args) {
        // Stack (legacy - ใช้ Deque แทนดีกว่า)
        Stack<Integer> stack = new Stack<>();
        stack.push(1);
        stack.push(2);
        stack.push(3);
        System.out.println("Stack: " + stack);
        System.out.println("Peek: " + stack.peek());   // ดูด้านบน
        System.out.println("Pop: " + stack.pop());     // ดึงออก
        System.out.println("Stack: " + stack);
        System.out.println("Empty: " + stack.isEmpty());
        
        // Deque แทน Stack (ดีกว่า - Java ไม่แนะนำ Stack class)
        Deque<Integer> deque = new ArrayDeque<>();
        deque.push(1);
        deque.push(2);
        deque.push(3);
        System.out.println("\nDeque as Stack: " + deque);
        System.out.println("Peek: " + deque.peek());
        System.out.println("Pop: " + deque.pop());
        
        // ใช้งาน Stack: ตรวจสอบ balanced parentheses
        System.out.println("\nBalanced parentheses:");
        System.out.println("'({[]})' = " + isBalanced("({[]})"));     // true
        System.out.println("'({[})' = " + isBalanced("({[})"));       // false
        System.out.println("'((()))' = " + isBalanced("((()))"));     // true
        System.out.println("')(' = " + isBalanced(")("));             // false
        
        // ใช้งาน Stack: Reverse a String
        String str = "Hello World";
        Deque<Character> charStack = new ArrayDeque<>();
        for (char c : str.toCharArray()) charStack.push(c);
        
        StringBuilder reversed = new StringBuilder();
        while (!charStack.isEmpty()) reversed.append(charStack.pop());
        System.out.println("\nReversed: " + reversed);
        
        // Evaluate Postfix Expression: "3 4 + 5 *" = (3+4)*5 = 35
        System.out.println("\nPostfix '3 4 + 5 *' = " + evaluatePostfix("3 4 + 5 *"));
    }
    
    static boolean isBalanced(String s) {
        Deque<Character> stack = new ArrayDeque<>();
        for (char c : s.toCharArray()) {
            if (c == '(' || c == '{' || c == '[') {
                stack.push(c);
            } else if (c == ')' || c == '}' || c == ']') {
                if (stack.isEmpty()) return false;
                char top = stack.pop();
                if ((c == ')' && top != '(') ||
                    (c == '}' && top != '{') ||
                    (c == ']' && top != '[')) return false;
            }
        }
        return stack.isEmpty();
    }
    
    static int evaluatePostfix(String expr) {
        Deque<Integer> stack = new ArrayDeque<>();
        for (String token : expr.split(" ")) {
            if (token.matches("-?\\d+")) {
                stack.push(Integer.parseInt(token));
            } else {
                int b = stack.pop(), a = stack.pop();
                switch (token) {
                    case "+" -> stack.push(a + b);
                    case "-" -> stack.push(a - b);
                    case "*" -> stack.push(a * b);
                    case "/" -> stack.push(a / b);
                }
            }
        }
        return stack.pop();
    }
}
```

---

## Queue และ Deque

```java
import java.util.*;

public class QueueDequeExample {
    public static void main(String[] args) {
        // Queue - FIFO (First In First Out)
        Queue<String> queue = new LinkedList<>();
        queue.offer("First");    // add (offer ไม่ throw exception)
        queue.offer("Second");
        queue.offer("Third");
        
        System.out.println("Queue: " + queue);
        System.out.println("Peek (front): " + queue.peek());   // ดูแต่ไม่เอาออก
        System.out.println("Poll (remove front): " + queue.poll());
        System.out.println("After poll: " + queue);
        
        // PriorityQueue - ดึงออกตาม priority (ค่าน้อยออกก่อน)
        PriorityQueue<Integer> pq = new PriorityQueue<>();
        pq.offer(5);
        pq.offer(1);
        pq.offer(3);
        pq.offer(2);
        pq.offer(4);
        
        System.out.print("\nPriority Queue (min first): ");
        while (!pq.isEmpty()) {
            System.out.print(pq.poll() + " ");
        }
        System.out.println();
        
        // Max heap
        PriorityQueue<Integer> maxPQ = new PriorityQueue<>(Collections.reverseOrder());
        maxPQ.addAll(Arrays.asList(5, 1, 3, 2, 4));
        System.out.print("Max PQ: ");
        while (!maxPQ.isEmpty()) System.out.print(maxPQ.poll() + " ");
        System.out.println();
        
        // ArrayDeque - double ended queue
        Deque<String> deque = new ArrayDeque<>();
        deque.addFirst("Middle");
        deque.addFirst("First");    // เพิ่มหัว
        deque.addLast("Last");      // เพิ่มท้าย
        
        System.out.println("\nDeque: " + deque);
        System.out.println("PeekFirst: " + deque.peekFirst());
        System.out.println("PeekLast: " + deque.peekLast());
        System.out.println("PollFirst: " + deque.pollFirst());
        System.out.println("PollLast: " + deque.pollLast());
        System.out.println("Deque: " + deque);
        
        // Simulate printer queue
        System.out.println("\n=== Printer Queue Simulation ===");
        Queue<String> printerQueue = new LinkedList<>();
        printerQueue.offer("Document1.pdf");
        printerQueue.offer("Report2023.docx");
        printerQueue.offer("Invoice.xlsx");
        printerQueue.offer("Photo.jpg");
        
        System.out.println("Jobs in queue: " + printerQueue.size());
        while (!printerQueue.isEmpty()) {
            String job = printerQueue.poll();
            System.out.println("Printing: " + job);
        }
        System.out.println("Queue empty: " + printerQueue.isEmpty());
    }
}
```

---

## Collections Utility Methods

```java
import java.util.*;

public class CollectionsUtils {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(Arrays.asList(5, 3, 8, 1, 9, 2, 7, 4, 6));
        
        // Sort
        Collections.sort(list);
        System.out.println("Sorted: " + list);
        
        // Reverse sort
        Collections.sort(list, Collections.reverseOrder());
        System.out.println("Reverse sort: " + list);
        
        // Binary search (ต้อง sort ก่อน ascending)
        Collections.sort(list);
        int idx = Collections.binarySearch(list, 7);
        System.out.println("BinarySearch 7: index " + idx);
        
        // Shuffle
        Collections.shuffle(list);
        System.out.println("Shuffled: " + list);
        
        // Min, Max
        System.out.println("Min: " + Collections.min(list));
        System.out.println("Max: " + Collections.max(list));
        
        // Frequency
        List<String> words = Arrays.asList("apple", "banana", "apple", "cherry", "apple");
        System.out.println("Frequency of 'apple': " + Collections.frequency(words, "apple"));
        
        // Reverse
        List<Integer> nums = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Collections.reverse(nums);
        System.out.println("Reversed: " + nums);
        
        // Fill
        List<String> filled = new ArrayList<>(Arrays.asList("a", "b", "c"));
        Collections.fill(filled, "x");
        System.out.println("Filled: " + filled);
        
        // Copy
        List<Integer> source = Arrays.asList(1, 2, 3);
        List<Integer> dest = new ArrayList<>(Arrays.asList(0, 0, 0));
        Collections.copy(dest, source);
        System.out.println("Copied: " + dest);
        
        // nCopies
        List<String> copies = Collections.nCopies(5, "Hello");
        System.out.println("nCopies: " + copies);
        
        // Unmodifiable
        List<Integer> original = new ArrayList<>(Arrays.asList(1, 2, 3));
        List<Integer> unmod = Collections.unmodifiableList(original);
        // unmod.add(4);  // UnsupportedOperationException!
        System.out.println("Unmodifiable: " + unmod);
        
        // Singleton
        List<String> single = Collections.singletonList("only");
        System.out.println("Singleton: " + single);
        
        // emptyList
        List<Object> empty = Collections.emptyList();
        System.out.println("Empty: " + empty);
        
        // Disjoint (ไม่มีสมาชิกร่วม)
        List<Integer> a = Arrays.asList(1, 2, 3);
        List<Integer> b = Arrays.asList(4, 5, 6);
        List<Integer> c = Arrays.asList(3, 4, 5);
        System.out.println("\nDisjoint a,b: " + Collections.disjoint(a, b));  // true
        System.out.println("Disjoint a,c: " + Collections.disjoint(a, c));  // false
    }
}
```

---

## โปรแกรมตัวอย่าง

### Student Grade Book

```java
import java.util.*;

public class GradeBook {
    
    record Student(String name, List<Integer> scores) {
        double average() {
            return scores.stream().mapToInt(Integer::intValue).average().orElse(0);
        }
        
        String grade() {
            double avg = average();
            if (avg >= 90) return "A";
            if (avg >= 80) return "B";
            if (avg >= 70) return "C";
            if (avg >= 60) return "D";
            return "F";
        }
    }
    
    public static void main(String[] args) {
        List<Student> students = new ArrayList<>();
        students.add(new Student("Alice", Arrays.asList(92, 88, 95, 90)));
        students.add(new Student("Bob", Arrays.asList(75, 82, 78, 80)));
        students.add(new Student("Charlie", Arrays.asList(65, 70, 68, 72)));
        students.add(new Student("Diana", Arrays.asList(98, 95, 97, 99)));
        students.add(new Student("Eve", Arrays.asList(55, 60, 58, 62)));
        
        // Print all
        System.out.println("╔══════════════════════════════════════════╗");
        System.out.printf("║ %-12s | %-8s | %-6s | Grade║%n", "Name", "Average", "Status");
        System.out.println("╠══════════════════════════════════════════╣");
        
        for (Student s : students) {
            double avg = s.average();
            String status = avg >= 60 ? "Pass  " : "Fail  ";
            System.out.printf("║ %-12s | %8.2f | %s | %-5s║%n",
                s.name(), avg, s.grade(), status);
        }
        System.out.println("╚══════════════════════════════════════════╝");
        
        // Sort by average
        students.sort((a, b) -> Double.compare(b.average(), a.average()));
        System.out.println("\nRanking:");
        for (int i = 0; i < students.size(); i++) {
            System.out.printf("%d. %s (%.2f)%n", 
                i + 1, students.get(i).name(), students.get(i).average());
        }
        
        // Statistics
        DoubleSummaryStatistics stats = students.stream()
            .mapToDouble(Student::average)
            .summaryStatistics();
        
        System.out.printf("\nClass Average: %.2f%n", stats.getAverage());
        System.out.printf("Highest: %.2f (%s)%n", 
            stats.getMax(), students.get(0).name());
        System.out.printf("Lowest: %.2f (%s)%n", 
            stats.getMin(), students.get(students.size()-1).name());
        
        // Count passing
        long passing = students.stream()
            .filter(s -> s.average() >= 60)
            .count();
        System.out.printf("Passing: %d/%d students%n", passing, students.size());
        
        // Grade distribution
        Map<String, Long> gradeDist = new LinkedHashMap<>();
        for (Student s : students) {
            gradeDist.merge(s.grade(), 1L, Long::sum);
        }
        System.out.println("\nGrade Distribution:");
        gradeDist.forEach((grade, count) -> 
            System.out.printf("  %s: %d students%n", grade, count));
    }
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Array Rotation

```java
// เฉลย
import java.util.Arrays;

public class ArrayRotation {
    // Rotate array left by k positions
    static int[] rotateLeft(int[] arr, int k) {
        int n = arr.length;
        k = k % n;  // ป้องกัน k > n
        int[] result = new int[n];
        for (int i = 0; i < n; i++) {
            result[i] = arr[(i + k) % n];
        }
        return result;
    }
    
    // Rotate array right by k positions
    static int[] rotateRight(int[] arr, int k) {
        int n = arr.length;
        k = k % n;
        return rotateLeft(arr, n - k);
    }
    
    public static void main(String[] args) {
        int[] arr = {1, 2, 3, 4, 5};
        System.out.println("Original: " + Arrays.toString(arr));
        System.out.println("Rotate left 2: " + Arrays.toString(rotateLeft(arr, 2)));
        System.out.println("Rotate right 2: " + Arrays.toString(rotateRight(arr, 2)));
    }
}
```

### แบบฝึกหัดที่ 2: ระบบคลังสินค้า (Inventory)

```java
// เฉลย
import java.util.*;

public class Inventory {
    record Product(String id, String name, int quantity, double price) {}
    
    static List<Product> products = new ArrayList<>();
    
    static void addProduct(String id, String name, int qty, double price) {
        products.add(new Product(id, name, qty, price));
        System.out.printf("เพิ่ม %s สำเร็จ%n", name);
    }
    
    static void displayAll() {
        System.out.println("\n=== รายการสินค้า ===");
        System.out.printf("%-8s %-20s %6s %10s %12s%n", 
            "ID", "ชื่อ", "จำนวน", "ราคา", "มูลค่า");
        System.out.println("-".repeat(60));
        double totalValue = 0;
        for (Product p : products) {
            double value = p.quantity() * p.price();
            totalValue += value;
            System.out.printf("%-8s %-20s %6d %10.2f %12.2f%n",
                p.id(), p.name(), p.quantity(), p.price(), value);
        }
        System.out.println("-".repeat(60));
        System.out.printf("%-42s มูลค่าทั้งหมด: %12.2f%n", "", totalValue);
    }
    
    public static void main(String[] args) {
        addProduct("P001", "Java Book", 100, 350.00);
        addProduct("P002", "Spring Boot Guide", 50, 450.00);
        addProduct("P003", "Docker Handbook", 75, 280.00);
        addProduct("P004", "Kubernetes Manual", 30, 520.00);
        
        displayAll();
        
        // Sort by price
        products.sort(Comparator.comparingDouble(Product::price));
        System.out.println("\nเรียงตามราคา:");
        products.forEach(p -> System.out.printf("  %s: %.2f%n", p.name(), p.price()));
        
        // Low stock alert
        System.out.println("\nสินค้าใกล้หมด (น้อยกว่า 50 ชิ้น):");
        products.stream()
            .filter(p -> p.quantity() < 50)
            .forEach(p -> System.out.printf("  ⚠️ %s: เหลือ %d ชิ้น%n", p.name(), p.quantity()));
    }
}
```

---

## สิ่งที่เรียนรู้ใน Part นี้

✅ Arrays พื้นฐาน  
✅ Multi-dimensional Arrays  
✅ Arrays Class utilities  
✅ ArrayList และการใช้งาน  
✅ LinkedList  
✅ Stack และ Deque  
✅ Queue และ PriorityQueue  
✅ Collections utility methods  

---

## ขั้นตอนต่อไป

**Part 007:** Object-Oriented Programming: Classes & Objects  
เราจะเรียนรู้:
- Classes และ Objects
- Constructors
- Encapsulation
- this keyword
- Enums
- Records

---

*Part 006 | Java & Spring Boot Course | สร้างโดย Claude Code*
