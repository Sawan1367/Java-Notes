# Pattern Printing
1. Divide it into a grid
2. Find the number of rows
3. Find the number of columns for each row
4. Analyze each row and find the mathematical relation between number of rows and number of columns (Also check if there is some symmetry or break pattern visible)
# Boxing in java
```java
import java.util.Arrays;
class Practice {
    public static void main(String[] args) {
        String str = "12345";
        String[] parts = str.split(""); // split by spaces
        int[] arr = Arrays.stream(parts).mapToInt(Integer::parseInt).toArray();
        for (int i : arr) {
            Integer boxed = i; // auto-boxing
            System.out.println(boxed.getClass().getName());
        }
    }
}
```
# Comparing Strings
1. `string1 == string2` ---> It compares the address of 2 strings
2. `string1.equals(string2)` ---> It compares the content of 2 strings, and it is Case-Sensitive
3. `string1.equalsIgnoreCase(string2)` ---> It compares the content of 2 strings, without concerning about the case-sensitivity of both the strings
# String Input
1. `sc.next()` ---> It takes input of one word. As soon as it finds a space - it stops taking the input
2. `sc.nextLine()` ---> It takes the entire line as input (until enter is hit)
# Prime Numbers from 1 to N
```java
public class SieveOfEratosthenes {
   public static void sieveOfEratosthenes(int n) {
       // Create a boolean array to mark prime numbers
       boolean[] isPrime = new boolean[n + 1];
       for (int i = 0; i <= n; i++) {
           isPrime[i] = true; // Initially, assume all numbers are prime
       }
       // Start marking multiples of each number as non-prime
       for (int p = 2; p * p <= n; p++) {
           if (isPrime[p]) { // If p is still marked as prime
               for (int i = p * p; i <= n; i += p) {
                   isPrime[i] = false; // Mark multiples of p as non-prime
               }
           }
       }
       // Print all prime numbers
       System.out.println("Prime numbers up to " + n + ":");
       for (int i = 2; i <= n; i++) {
           if (isPrime[i]) {
               System.out.print(i + " ");
           }
       }
   }
   public static void main(String[] args) {
       int n = 50; // Example: Find primes up to 50
       sieveOfEratosthenes(n);
   }
}
```
