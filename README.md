# KqlSieveOfEratosthenes
a fun challenge for a KQL version of the Sieve of Eratosthenes

[Original post](https://www.linkedin.com/posts/sloutsky_azure-data-explorer-activity-6914950553500291072-iQuu?utm_source=linkedin_share&utm_medium=member_desktop_web) by Alexander Sloutsky
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

Here is my approach. I am clocking in 4.22 sec on a dev sku cluster (have not spun up my free cluster yet :( )

``` kusto
// Added the Sieve of Eratosthenes optimization to the script to minimize the list of dividers
let n = int(1000000);
let rootOfn = toint(sqrt(n)); // Need the root of "n" to know when to stop counting prime dividers
let Optimize = range i from 2 to rootOfn step 1; // Building a separate range that stops using the variable above
let OptimizerArray = // this will build an array that contains the optimized list of primes we can divide by. We're casting it as a scalar to use it later in our script 
        toscalar (
                    Optimize
                    | where (i == 3  or i % 3  != 0) // The following where clauses removes the usual suspects from our list
                        and (i == 5  or i % 5  != 0)
                        and (i == 7  or i % 7  != 0)
                    | summarize make_set(toint(i)) // We finish off by summarizing and creating a bag
                 );
range num from 3 to n step 1 // 
| where     num % 2 != 0  // pruning the list starting with 2 and stoping when perf does not gain any more 
        and num % 3 != 0
        and num % 5 != 0
        and num % 7 != 0
        and num % 9 != 0
        and num % 11 != 0 // Negligable return on perf past that point
| extend dividers = OptimizerArray
| mv-apply dividers to typeof(long) on
(
    summarize Dividers=countif(num % dividers == 0 and num != dividers)
)
| where Dividers == 0
| union (print num=2) // '2' is prime number too!
| count
``` 
