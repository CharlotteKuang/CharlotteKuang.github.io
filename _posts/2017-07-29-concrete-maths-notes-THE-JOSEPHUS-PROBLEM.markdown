---
layout: post
title:  "Notes of Concrete Maths--The Josephus Problem"
date:   2017-07-29 16:27:00 +0800
categories: maths
---

The [Josephus problem](https://en.wikipedia.org/wiki/Josephus_problem) is a problem where N people standing in the cirle and startint to count from 1 to K, and counting out the Kth people in the precedure. The counting restart on the next person starting from 1 again until there is only one person left in the circle.

The book presents a trivial case where K = 2. And here J(N) is used to respresnt the last person in the N-people circle.

![Alt text]({{ site.url }}/assets/circle1.png)

As in the example shown by the picture above, person 2, 4, 6 are to be counted out at the first round.

At this point, let's look at the general case after the first counting round. And the case would be different concerning whether N is an even or odd number.

<table border="1">
    <tr><th colspan="2">The Even case(2n people in the circle):</th></tr>
    <tr>
        <td>old index</td>
        <td>new index</td>
    </tr>
    <tr>
        <td>1</td>
        <td>1</td>
    </tr>
    <tr>
        <td>3</td>
        <td>2</td>
    </tr>
    <tr>
        <td>5</td>
        <td>3</td>
    </tr>
    <tr>
        <td>2n-1</td>
        <td>n</td>
    </tr>
</table>

From the above table, we assume that J(2n) is the index of last person in the 2N-people circle and J(n) is the index of last person in the N-people circle. Therefore, we must be sure that 2J(n)-1 = J(2n) because of the mapping between two circles.

<table border="1">
    <tr><th colspan="2">The Odd case(2n+1 people in the circle):</th></tr>
    <tr>
        <td>old index</td>
        <td>new index</td>
    </tr>
    <tr>
        <td>3</td>
        <td>1</td>
    </tr>
    <tr>
        <td>5</td>
        <td>2</td>
    </tr>
    <tr>
        <td>2n-1</td>
        <td>n-1</td>
    </tr>
    <tr>
        <td>2n+1</td>
        <td>n</td>
    </tr>
</table>

Similarly, we can have 2J(n)+1 = J(2n+1) when N is odd.

Therefore, we have some induction like:

<p style='text-align:center;'>J(1) = 1,</p>

<p style='text-align:center;'>J(2n) = 2J(n) - 1,      for n >= 1</p>

<p style='text-align:center;'>J(2n + 1) = 2J(n) + 1,  for n >= 1</p>

Let's calculate some cases to guess the pattern of the solution to this problem:

<table border="1">
    <tr>
        <td> n </td>
        <td> 1 </td>
        <td> 2 </td>
        <td> 3 </td>
        <td> 4 </td>
        <td> 5 </td>
        <td> 6 </td>
        <td> 7 </td>
        <td> 8 </td>
        <td> 9 </td>
        <td> 10 </td>
        <td> 11 </td>
        <td> 12 </td>
        <td> 13 </td>
        <td> 14 </td>
        <td> 15 </td>
        <td> 16 </td>
    </tr>
    <tr>
        <td> J(n) </td>
        <td> 1 </td>
        <td> 1 </td>
        <td> 3 </td>
        <td> 1 </td>
        <td> 3 </td>
        <td> 5 </td>
        <td> 7 </td>
        <td> 1 </td>
        <td> 3 </td>
        <td> 5 </td>
        <td> 7 </td>
        <td> 9 </td>
        <td> 11 </td>
        <td> 13 </td>
        <td> 15 </td>
        <td> 1 </td>
    </tr>
</table>

> It seems we can group by powers of 2 (marked by vertical lines in the table); J(n) is always 1 at the beginning of a group and it increases by 2 within a group. So if we writen in the form n = 2m + l,where 2m is the largest power of 2 not exceeding n and where l is what's left, the solution to our recurrence seems to be:

<p style='text-align:center'>J(2<sup>m</sup> + l) = 2*l + 1</p>
