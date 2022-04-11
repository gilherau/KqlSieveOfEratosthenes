# Counting primes in KQL
This is my response to a fun challenge for a KQL version of the Sieve of Eratosthenes. Challenge accepted!

## Original post

[Initial challenge](https://www.linkedin.com/posts/sloutsky_azure-data-explorer-activity-6914950553500291072-iQuu?utm_source=linkedin_share&utm_medium=member_desktop_web) by Alexander Sloutsky
``` kusto
range num from 3 to 50000 step 1 
| where num % 2 != 0 // skip even numbers
| extend divider = range(3, num/2, 2) // divider candidates
| mv-apply divider to typeof(long) on
(
  summarize Dividers=countif(num % divider == 0) // count dividers
)
| where Dividers == 0 // prime numbers don't have dividers
| union (print num=2) // '2' is prime number too!
| count
``` 
## My approach
I am clocking in 12 sec on a dev sku cluster (have not spun up my free cluster yet :( )

###todos:
- Hygiene with data types to accomodate bigger numbers :)
- Generalize some of the code to be functions if possible


``` kusto
// Added the Sieve of Eratosthenes optimization to the script to minimize the list of dividers
let n = int(1000000);
let rootOfn = toint(sqrt(n)); // Need the root of "n" to know when to stop counting prime dividers
let Optimize = range i from 2 to rootOfn step 1; // Building a separate range that stops using the variable above
let RawOptimizerArray = // this will build an array that contains a contrained list of integers that we must indentify as primes we can divide by. We're casting it as a scalar to use it later in our script 
        toscalar (
                    Optimize
                    | summarize make_set(toint(i)) // We finish off by summarizing and creating a bag
                 );
let OptimizerSet = // Now we will use the MV-apply trick to remove the non-primes and create an array of primes up to RootOfN
    toscalar ( // using toscalar to convert our final array resultset to a scalar value for later use
                Optimize
                | extend dividers = RawOptimizerArray // grabbing our list of integers of interest
                | mv-apply dividers to typeof(long) on 
                    (
                        summarize Dividers=countif(i % dividers == 0 and i != dividers) // using the mv-apply trick to find the primes within that constrained list
                    )
                | where Dividers == 0 // keeping only the primes
                | summarize make_set(i) //making an array out of our results
             );
range num from 2 to n step 1 // Build the final list of integers.
| extend dividers = OptimizerSet //grabbing our optimizer set from before
| mv-apply dividers to typeof(long) on
(
    summarize Dividers=countif(num % dividers == 0 and num != dividers) // using the mv-apply trick to find the primes
)
| where Dividers == 0 //keeping only the primes
| count //final count
``` 
