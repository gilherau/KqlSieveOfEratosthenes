# Counting primes in KQL

Counting primes in a new language you may not be familiar with is always fun and a great learning experience. I encourage you to read the original challenge from Alexander (See below) and try it yourself before you look at the solution I came up with. 

## Original post

>Want to play with #BigData platform that powers #Microsoft, and win quality time with the team who built it? How about a small fun contest then: how efficient can you count prime numbers with #KQL? The winner gets to meet #Kusto engineering team online and for a Q&A session!

>To get you started, here is a small query that works in somewhat naïve way. Can you improve it and make it run faster? Once you’re happy with the result: find out how many prime numbers are there under 1,000,000?

>If you don’t have your own #Kusto/#ADX cluster already - you can get it #free here: https://aka.ms/kustofree

[link to initial challenge](https://www.linkedin.com/posts/sloutsky_azure-data-explorer-activity-6914950553500291072-iQuu?utm_source=linkedin_share&utm_medium=member_desktop_web) by Alexander Sloutsky
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
I am clocking in 6 sec on a dev sku cluster (have not spun up my free cluster yet :( )

### todos:
- Hygiene with data types to accomodate bigger numbers :) (done)
- Generalize some of the code to be functions if possible (done)
- Pre-hydrate list of prime dynamically as parameters


``` kusto
// Added the Sieve of Eratosthenes optimization to the script to minimize the list of dividers
let n = long(1000000); //setting the number from which we are counting down the prime from as variable "n"
//Begin OptimizeSieve(n) function to return an optimized list of dividers to cut down on running time.
let OptimizeSieve = (n: long) 
{ 
let Optimize = range i from 2 to sqrt(n) step 1; // Building a range of integers that stops at "rootOfN"
let RawOptimizerArray = // this will build an scalar array that contains a contrained list of integers that we must indentify as primes we can divide by. We're casting it as a scalar to use it later in our script 
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
OptimizerSet             
}; 
range num from 3 to n step 2 // Build the final list of integers. Note we are starting from an odd number and using a step value of 2 which lets us skip all even numbers past the number 2 which we artificially re-inject later for an accurate count 
| extend dividers = OptimizeSieve(n) // invoking the function that returns the optimized list of dividers
| mv-apply dividers to typeof(long) on
(
    summarize Dividers=countif(num % dividers == 0 and num != dividers) // using the mv-apply trick to find the primes
)
| where Dividers == 0 //keeping only the primes
| union (datatable(num:long) [2]) // adding the special case of 2 back in the dataset
| count //final count
``` 
