# How do I explain things

[TOC]

## Python: explaining higher order function.

### Question:

Why is f not just lambda x: pow( x, 2) ?

![img](https://d1b10bmlvqabco.cloudfront.net/attach/ixj56cq4o911qa/iy25yohurv0206/iz9gh1fmn52r/Screen_Shot_20170216_at_10.49.38_PM.png)

### 	Answer:

​		Good question. 

​		If you look carefully in repeat_sum, the next number = f(previous number), so we need to take the square root of previous number, and then plus one, and then square it. 

​		Consider write it out:

```python
#Whenever we are applying f:
sum_squares(3) # 1**2 + 2**2 + 3**2
#We have:
f(1) = (1**(1/2) + 1)**2 = 2**2
f(4) = (4**(1/2) + 1)**2 = 3**2
```





## Explaining algorithms/data structure question (Amortized Runtime Analysis):

### Question:

>Amortized Runtime Analysis
>
>I have a couple questions about amortized runtime analysis because I don't get it fully.
>
>Why do we do amortized analysis?
>
>Is there amortized runtime analysis for nonconstant functions?? What does it mean for us to have an ai such that phi(i) stays ahead of ci so that phi(i) is always greater than or equal to 0? I thought that if phi(i) always stayed ahead of ci then ai-ci would end up being negative, because eventually phi(i) may surpass phi(i+1).
>
>This mainly has to do with lecture #19, the third asymptotics lecture.

### Answer:

1.Why do we do amortized analysis?

 

It's really depending on the context, see from the optional textbook *:

> ## Coping with dependence on inputs.
>
>  For many problems, the running time can vary widely depending on the input.
>
>  
>
> - *Input models.* We can carefully model the kind of input to be processed. This approach is challenging because the model may be unrealistic. 
> - *Worst-case performance guarantees.* Running time of a program is less than a certain bound (as a function of the input size), no matter what the input. Such a conservative approach might be appropriate for the software that runs a nuclear reactor or a pacemaker or the brakes in your car. 
> - *Randomized algorithms.* One way to provide a performance guarantee is to introduce randomness, e.g., quicksort and hashing. Every time you run the algorithm, it will take a different amount of time. These guarantees are not absolute, but the chance that they are invalid is less than the chance your computer will be struck by lightning. Thus, such guarantees are as useful in practice as worst-case guarantees. 
> - *Amortized analysis.* For many applications, the algorithm input might be not just data, but the sequence of operations performed by the client. Amortized analysis provides a worst-case performance guarantee on **a sequence of operations.**

Moreover, amortized analysis is really an unique way of analysis. "It counts all steps together instead of bounding each step separately"**:

> ...This is known as an “amortized” analysis because we spread the cost of the few expensive operations, by assigning a portion of it to each of a large number of inexpensive operations.  

2. Is there amortized runtime analysis for nonconstant functions?

   ​

I can't find an example: after a while of searching, I found all instances of amortized runtime (fibonacci heaps, Dynamic Arrays, Queue, WQUPC...) are constant. 



3. What does it mean for us to have an ai such that phi(i) stays ahead of ci so that phi(i) is always greater than or equal to 0? I thought that if phi(i) always stayed ahead of ci then ai-ci would end up being negative, because eventually phi(i) may surpass phi(i+1).

 

Note that Φ(⋅) is the potential. It depends on previous potential, current real cost, and ai of which you "deposit". If you go to the next slide you can run-through the example. Another way of interpreting equation is that $ai=ci+Φi+1−Φi⟹ΔΦ=Φi+1−Φi=ai−ci$–which is presented in "change in potential" row. $ai−ci$ is the change in potential. We are actually fine that the change being negative for few cases: i=4,8. Note that the only thing that we are "in control" of this analysis is ai, not ci. 

 

![img](https://d1b10bmlvqabco.cloudfront.net/attach/ir6ikxxrjtm3j5/is25wwpv21b3nn/j0rtr2ml8hbx/Screen_Shot_20170327_at_1.01.03_AM.png)

 

\*: *Algorithms, 4th Edition* by Robert Sedgewick and Kevin Wayne

\*\*: *Introduction to Algorithms: A Creative Approach* by Udi Manber