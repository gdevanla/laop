# laop - Linear Algebra of Programming library

The LAoP discipline generalises relations and functions treating them as
Boolean matrices and in turn consider these as arrows.

__LAoP__ is a library for algebraic (inductive) construction and manipulation of matrices
in Haskell. See [my Msc Thesis](https://github.com/bolt12/master-thesis) for the
motivation behind the library, the underlying theory, and implementation details.

This module offers many of the combinators mentioned in the work of
[Macedo (2012)](https://repositorium.sdum.uminho.pt/handle/1822/22894) and [Oliveira (2012)](https://pdfs.semanticscholar.org/ccf5/27fa9179081223bffe8067edd81948644fc0.pdf). 

See the candidate package in hackage [here](https://hackage.haskell.org/package/laop-0.1.0.0/candidate)

## Features

This library offers:

- One that uses normal datatypes (`Void`, `()`, `Either`) for matrix dimensions and two
  type families `FromNat` and `Count` to make it easier to work with these type of matrices.
- Other that uses type level naturals for the dimensions and is just a newtype wrapper
  around the other matrix data type.
- A very simple Probabilistic Programming module and several functions and data types to
  make it easier to deal with sample space.

Given this, this new matrix formulation compared to other libraries one has much more advantages, such as:
        
- Has an inductive definition enabling writing matrix manipulation functions in a much more elegant and calculational way;
- Statically typed dimensions;
- Polymorphic data type dimensions;
- Polymorphic matrix content;
- Awesome dimension type inference;
- Fast type natural conversion via `FromNat` type family;
- Matrix `Junc` and `Split`-ing in O(1);
- Matrix composition takes advantage of divide-and-conquer and fusion laws.
        
Unfortunately, this approach does not solve the issue of type dimensions being in some way constrainted making it impossible to write Arrow instances, for example. Type inference isn't perfect, when it comes to infer the types of matrices which dimensions are computed using type level naturals multiplication, the compiler needs type annotations in order to succeed.

## Notes

This is still a work in progress, any feedback is welcome!

## Example

```Haskell
{-# LANGUAGE DeriveGeneric #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE TypeOperators #-}
{-# LANGUAGE TypeApplications #-}
{-# LANGUAGE DataKinds #-}

module Main where

import Matrix.Type
import Utils
import Dist
import GHC.TypeLits
import Data.Coerce
import GHC.Generics
import Control.Category hiding (id)
import Prelude hiding ((.))

-- Monty Hall Problem
data Outcome = Win | Lose
    deriving (Bounded, Enum, Eq, Show, Generic)

switch :: Outcome -> Outcome
switch Win = Lose
switch Lose = Win

firstChoice :: Matrix Double () Outcome
firstChoice = col [1/3, 2/3]

secondChoice :: Matrix Double Outcome Outcome
secondChoice = fromF' switch 

-- Dice sum

type SS = Natural 1 6 -- Sample Space

sumSS :: SS -> SS -> Natural 2 12
sumSS = coerceNat (+)

sumSSM = fromF' (uncurry sumSS)

condition :: (Int, Int) -> Int -> Int
condition (fst, snd) thrd = if fst == snd
                               then fst * 3
                               else fst + snd + thrd

conditionSS :: (SS, SS) -> SS -> Natural 3 18
conditionSS = coerceNat2 condition

conditionalThrows = fromF' (uncurry conditionSS) . khatri (khatri die die) die

die :: Matrix Double () SS
die = col $ map (const (1/6)) [nat @1 @6 1 .. nat 6]

-- Sprinkler

rain :: Matrix Double () Bool
rain = col [0.8, 0.2]

sprinkler :: Matrix Double Bool Bool
sprinkler = fromLists [[0.6, 0.99], [0.4, 0.01]]

grass :: Matrix Double (Bool, Bool) Bool
grass = fromLists [[1, 0.2, 0.1, 0.01], [0, 0.8, 0.9, 0.99]]

state :: Matrix Double () (Bool, (Bool, Bool))
state = khatri grass identity . khatri sprinkler identity . rain

grass_wet :: Matrix Double (Bool, (Bool, Bool)) One
grass_wet = row [0,1] . kp1

rainning :: Matrix Double (Bool, (Bool, Bool)) One
rainning = row [0,1] . kp2 . kp2 

main :: IO ()
main = do
    putStrLn "Monty Hall Problem solution:"
    prettyPrint (secondChoice . firstChoice)
    putStrLn "\n Sum of dices probability:"
    prettyPrint (sumSSM `comp` khatri die die)
    putStrLn "\n Conditional dice throw:"
    prettyPrint conditionalThrows
    putStrLn "\n Checking that the last result is indeed a distribution: "
    prettyPrint (bang . sumSSM . khatri die die)
    putStrLn "\n Probability of grass being wet:"
    prettyPrint (grass_wet . state)
    putStrLn "\n Probability of rain:"
    prettyPrint (rainning . state)
```

```Shell
Monty Hall Problem solution:
┌                    ┐
│ 0.6666666666666666 │
│ 0.3333333333333333 │
└                    ┘

 Sum of dices probability:
┌                       ┐
│ 2.7777777777777776e-2 │
│  5.555555555555555e-2 │
│  8.333333333333333e-2 │
│    0.1111111111111111 │
│    0.1388888888888889 │
│   0.16666666666666666 │
│    0.1388888888888889 │
│    0.1111111111111111 │
│  8.333333333333333e-2 │
│  5.555555555555555e-2 │
│ 2.7777777777777776e-2 │
└                       ┘

 Conditional dice throw:
┌                       ┐
│ 2.7777777777777776e-2 │
│  9.259259259259259e-3 │
│ 1.8518518518518517e-2 │
│  6.481481481481481e-2 │
│  5.555555555555555e-2 │
│  8.333333333333333e-2 │
│   0.12962962962962962 │
│    0.1111111111111111 │
│    0.1111111111111111 │
│   0.12962962962962962 │
│  8.333333333333333e-2 │
│  5.555555555555555e-2 │
│  6.481481481481481e-2 │
│ 1.8518518518518517e-2 │
│  9.259259259259259e-3 │
│ 2.7777777777777776e-2 │
└                       ┘

 Checking that the last result is indeed a distribution: 
┌     ┐
│ 1.0 │
└     ┘

 Probability of grass being wet:
┌                    ┐
│ 0.4483800000000001 │
└                    ┘

 Probability of rain:
┌     ┐
│ 0.2 │
└     ┘
```
