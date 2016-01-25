---
layout: post
title:  "Haskell Evolution"
date:   2016-01-24 08:50:00 -0500
visible: 0
---
## Implementing the Vigenere Cipher

This post is intended to be an investigation into improving my Haskell style. The following is an assignment for an undergrad-level Introduction to Security course, but I think it may well serve a dual role as a basis for improving a skillset (what code doesn't do that?).

My initial thoughts are that I should be using more composition than I am, and that I might be in a monoid or semi-group with the Vigenere operation? And surely there's a better (more concise) way to write the main method...

(24 January 2016)

- - -

### The Problem

> The task of this exercise is to implement a polyalphabetic cryptosystem (specifically the Vignere Cipher). Your program must accept a key, an action (either "encrypt" or "decrypt"), and either the plaintext (and output the cipher text) or the ciphertext (and output the plaintext).
>
> . . .
> 
> You must remove any non-alphabetic characters from the key and the source data (i.e. purge white-space, punctuation, and numerals). Also, all lowercase characters must be converted to upper-case.
> 
> . . .
> 
> #### **Command-Line Version**
> 
> A simple command-line program that accepts four arguments via the command-line.
> 
> The first argument is the action. It will be a string of either "encrypt" or "decrypt".
> 
> The second argument is a filename to the key. Your program will open this filename, and use its contents as the key to the polyalphabetic cipher.
> 
> The third argument is a filename containing the source/input data. The fourth argument will be a filename to which your program will write the output. Your program will read the contents of the source file, perform the action specified in the first argument using the key in the second argument, and output the data to the destination file specified in the fourth argument.
> 
> . . .
>
> #### **Programming Languages**
>
> You may use C, C++, C#, Java, HTML/JavaScript, Python, or PHP. If you wish to use another language, please get written approval from me prior to submission!
> 
>. . .
>
> #### **Grading Criteria**
> 
> Your program will be graded on several criteria:
> 
> - Successful compilation; Successful execution (no crashing!)
> - Correct implementation of the algorithm
> - Correct output
> - Source-code commenting - explain to me what is happening at the major steps of your program


- - - 

### Solution - Version 1.0

#### The Main Method

The main module contains the IO monad for interacting with the user's call. It does some basic error handling, parses the input, and outputs the decrypted or encrypted text as required. It calls on the functions provided by the Vigenere module, as shown below.

{% highlight haskell linenos %}
module Main where

import Vigenere

import System.IO
import System.Environment
import Control.Exception

main :: IO ()
main = do
  let message = "Program syntax: [./]assignment2[.exe] { encrypt | decrypt } /path-to/key.txt /path-to/source.txt /path-to/output.txt"
  args <- getArgs
  case length args of -- We make sure the user passed in exactly four arguments, else exit gracefully
    4 -> do
           let [mode, key, source, out] = args
           case (mode == "encrypt" || mode == "decrypt") of -- We make sure the user specified a valid -crypt mode
             True -> do -- We try to read the key text file. This either returns the String text or throws an exception
                       res <- try $ readFile key :: IO (Either SomeException String)
                       case res of -- On error, exit gracefully.
                         Left e -> do putStr "Error parsing key text: "; print e
                         Right keyTxt -> do -- eg: concat [1,2,3] [4,5,6] = [1,2,3,4,5,6]
                                           let key' = concat $ lines keyTxt
                                           case (length key' > 0) of
                                             True -> do
                                                       res' <- try $ readFile source :: IO (Either SomeException String)
                                                       case res' of
                                                         Left e -> do putStr "Error parsing source file: "; print e
                                                         Right sourceTxt -> do
                                                                              let source' = concat $ lines sourceTxt
                                                                              let crypted = runVig mode key' source'
                                                                              res'' <- try $ writeFile out crypted :: IO (Either SomeException ())
                                                                              case res'' of
                                                                                Left e -> do putStr "Error writing output: "; print e
                                                                                Right _ -> putStrLn $ "Successful " ++ mode ++ "ion!"
                                             _ -> putStrLn "Blank key! Exiting..."
             _ -> do putStrLn $ "Unrecognized encrypt/decrypt selection: " ++ mode; putStrLn message
    _ -> do putStrLn "Invalid number of arguments provided."; putStrLn message
{% endhighlight %}

#### The Vigenere Implementation

I can probably clean this up considerably, but it works and is fairly concise. Not as concise as the two-line implementation in the Haskell Vigenere library, but...

We rely on the so-called "plaintext" files to be Unicode encoded, and work primarily with the numeric value of the Unicode representation in order to perform the required addition and subtraction. 

{% highlight haskell linenos %}
module Vigenere where

import Data.List
import Data.Char

-- ========================================================================== --
-- Notes on my comments:
--   |y| means the numeric value of the character y
--   e.g. |A| == 65 == fromEnum 'A'
--     This value is used as the "base" value of the start of the alphabet.
-- ========================================================================== --

base = fromEnum 'A'

-- Apply the Vigenere cipher to a single char with a key char:
-- Decrypt:
--   If |key| <= |plain text| then return |plain text| - |key| + |A| as a char.
--   Else, return (26 - |key| - |A|) + (|plain text| - |A|) + |A| as a char
decrypt :: Char -> Char -> Char
decrypt key txt | k <= p    = chr $ (p - k) + base
                | otherwise = chr $ res + base
  where
    k = (fromEnum key) - base
    p = (fromEnum txt) - base
    res = (26 - k) + p

-- Encrypt:
--   Essentially, the encrypted char is the key char value +shifted by the
--     plain text char value, all modded by 26 to keep it in the alphabet.
encrypt :: Char -> Char -> Char
encrypt key txt = chr $ ((k + p) `mod` 26) + base
  where
    k = (fromEnum key) - base
    p = (fromEnum txt) - base

-- ========================================================================== --
-- Some helper functions...
-- Take a list of length <= n, an arbitrary length n, and repeat the list out
--   to contain at least n elements. As long as the key list is longer than
--   the list to crypt, we're good, because the zipWith function stops at the
--   end of the shortest length list.
repLen :: [a] -> Int -> [a]
repLen [] _ = []
repLen l n | n > length l = repLen (l ++ l) n
           | otherwise    = l

-- Clean up the line from the file and return all uppercase characters. Lines
--   do not contain newlines thanks to "lines" call in main. "isLetter" returns
--   true if the char is in one of the categories defined by Unicode standard
--   available at: <www.unicode.org/reports/tr44/tr44-14.html#GC_Values_Table>
-- Note the use of Lambda notation in order to apply list comprehension (quasi-
--   set-builder notation) to a list. Kind of equivalent to filtering a list
--   l by the condition that elements in l are letters, then returning the
--   uppercase version of the letter.
cleanLine :: [Char] -> [Char]
cleanLine = \l -> [toUpper x' | x' <- l, isLetter x' == True]

-- Run application of Vigenere Cipher in a specified mode with a key string,
--   a plaintext string, and output a en/de-crypted string. Note that a String
--   is the same as a list of chars. The "zipWith" function takes two lists
--   and a function, applies the function with an element of each list, and
--   outputs the list of resultant elements.
--   e.g. zipWith (+) [1,2,3] [4,5,6] outputs [5,7,9]
runVig :: String -> [Char] -> [Char] -> [Char]
runVig mode key txt = (\f -> zipWith f
                             (cleanLine $ repLen key $ length $ cleanLine txt)
                             (cleanLine txt))
                             (if mode == "decrypt" then decrypt else encrypt)
{% endhighlight %}
