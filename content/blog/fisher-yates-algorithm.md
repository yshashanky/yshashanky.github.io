---
title: "Fisher-Yates algorithm"
date: 2024-09-03
description: "A look at the Fisher-Yates shuffle algorithm, also known as the Knuth shuffle and a simple JavaScript implementation to get a random subset from an array."
draft: false
---

Came across this algorithm while exploring how to randomize data in an array or list, and it is working quite efficiently and accurately. It is also known as the Knuth shuffle, developed by Ronald Fisher and Frank Yates in 1938.

### How it works

This algorithm works by iterating over the elements of the array in reverse order and swapping each element with a randomly selected element that comes before it in the array. To find the random index, I am multiplying the index by the output of the `Math.random()` function. This process ensures that each element has an equal probability of ending up in any position in the shuffled array.

I am using this to get a subset of the random array. Below is a simple implementation of the Fisher-Yates algorithm in JavaScript:

```js
function getRandomSubset(arr, length) {
  const copyArray = arr.slice();

  // Shuffle the array using the Fisher-Yates algorithm
  for (let i = copyArray.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [copyArray[i], copyArray[j]] = [copyArray[j], copyArray[i]];
  }

  return copyArray.slice(0, length);
}
```

### Walkthrough

In this implementation:

1. We start from the last element of the array and iterate backward (`let i = copyArray.length - 1; i > 0; i--`).
2. For each element at index `i`, we generate a random index `j` using the function `Math.floor(Math.random() * (i + 1))` such that `0 <= j <= i`.
3. We then swap the elements at indices `i` and `j`.
4. This process continues until we reach the first element of the array, resulting in a shuffled array.
5. Additionally, I am slicing the random array to the required length.

I hope it helps, and that's all for today. Until next time, take care and have a great time!
