---
title: "Padding Oracle Attack"
date: 2023-07-24T07:10:07-04:00
draft: false
---

This post is an effort to explain the padding oracle attack for Cipher Block Chaining (CBC) mode of encryption. In a successful attack, an attacker can recover the plaintext without any knowledge of the key. If the application, which is handling the decryption, leaks information about the valid and invalid padding, an attacker can send specially crafted ciphertext blocks and recover the plaintext. Below are the detials of the attack. 
 

## Padding
First, let's discuss how padding works. As CBC is a block cipher, data is encrypted in blocks and all the blocks are of equal size. Before encryption, the plaintext is divided into blocks, and because the plaintext can be of arbitrary length, we need to add padding to match the block size. In PKCS#5, the final block of plaintext is padded with N bytes of value N. 
\
\
For example, for an 8 byte block, if the plaintext is of 3 bytes, 5 bytes of padding will be added (green is plaintext and yellow is padding):
\
![padding1](/images/padding-oracle/image1.png)

Similarly, for 6 bytes of plaintext, 2 bytes of padding will be added:

![padding2](/images/padding-oracle/image2.png)

Notice that for 5 bytes padding, we added 05 and for 2 bytes padding we added 02 (it’s in hexadecimal). What if plaintext is size of a block? We will still add padding by adding another block as shown below:

![padding3](/images/padding-oracle/image3.png)

## CBC Decryption Mode
Below figure describes CBC mode decryption (source: [Wikipedia](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation##/media/File:CBC_decryption.svg)):

![decryption4](/images/padding-oracle/image4.png)

The receiver will use CBC decryption to obtain the plaintext + padding. The receiver will then remove padding to recover the original message.  But if the receiver finds final N bytes not equal to N, it will return an error, we will call it **error_padding**. For example, the following block will return an error:

![decryption5](/images/padding-oracle/image5.png)


In above block, the 03 indicates that the padding is of size 3 bytes, but only 2 bytes were found.
\
\
And if the receiver receives an invalid cipher text and valid padding, it will also return an error, let’s call it **error_cipher**.
\
\
In a padding oracle attack, if an attacker can detect these errors, he can recover plaintext! 

## Situation
We assume that attacker has access to ciphertexts C1 and C2, and he is trying to recover the plaintext (P2):

![situation6](/images/padding-oracle/image6.png)

Originally, suppose that the values of C1, I2, and P2 were as below (in hexadecimal):
\
C1 =
![situation7](/images/padding-oracle/image7.png)

I2 = 
![situation8](/images/padding-oracle/image8.png)

P2 = C1 ⊕ I2 = 
![situation9](/images/padding-oracle/image9.png)

We can see that P2 has 3 bytes of padding. Attacker does not have knowledge of I2 and P2, he will only modify C1 and try to recover P2. 

## The Attack
Attacker will try to cause errors by modifying the cipher text, he especially wants to see **error_padding**. To do so, attacker will start modifying the C1 bytes starting from the first byte, e.g., attacker will send the following C1 (red part is the modified one):

![attack10](/images/padding-oracle/image10.png)

This will cause an **error_cipher** but not **error_padding** as padding bytes are still not changed, attacker will continue changing bytes until it reaches 6th byte:

![attack11](/images/padding-oracle/image11.png)

This will cause an **error_padding**, because the XOR of 6th byte (A1) with 6th byte of I2 (06) is not 03. Attacker learned from this error that there is 3 bytes of padding, or the last three bytes of plaintext is 03. 
\
\
Using this information, attacker can find out final 3 bytes of I2, for example, for right most byte (using unmodified C1 here):

90 ⊕ I2 (8th byte) = 03 
\
We can say: I2 (8th byte) = 90 ⊕ 03 = 93
\
Similarly: I2 (7th byte) = 1C ⊕ 03 = 1F
\
And: I2 (6th byte) = 05 ⊕ 03 = 06
\
\
So attacker knows following about the I2 (X are unknown values):

![attack12](/images/padding-oracle/image12.png)

Next, attacker wants to increase the padding bytes to 4, i.e., final four bytes of plaintext should be 04. Since attacker already knows last 3 bytes of I2, he can easily make that last three bytes of P2 to be 04. 
\
\
Attacker wants this: 
8th byte of plaintext = 04 = C1 (8th byte) ⊕ I2 (8th byte)
\
\
Since he knows I2:
\
C1 (8th byte) ⊕ 93 = 04
\
Yields to: C1 (8th byte) = 93 ⊕ 04 = 97
\
\
Similarly, attacker will calculate 7th and 6th byte of C1 to make the plaintext 04:
\
C1 (7th byte) = 1F ⊕ 04 = 1B
\
C1 (6th byte) = 06 ⊕ 04 = 02
\
\
Now, the C1 will look like this (blue are the changed bytes):

![attack13](/images/padding-oracle/image13.png)


Which will produce a P2 with last 3 bytes equal to 04, but the receiver will return an **error_padding** because the 5th byte of P2 is not 04 and receiver is expecting 4 bytes of padding. Attacker doesn’t know 5th byte of I2, so he will try different values (brute force) of C1 at 5th byte until the receiver doesn’t respond with an **error_padding** or the 5th byte of P2 is 04.  After brute forcing 5th byte of C1, attacker will not get any error for the below payload:

![attack14](/images/padding-oracle/image14.png)

Since 1A is producing 04 at 5th byte of P2, padding will be fine. Attacker can now calculate 5th byte of I2:
\
Since 1A = I2 (5th byte) ⊕ 04
\
Yields to: I2 (5th byte) = 1A ⊕ 04 = 1E 
\
\
You can see in the original I2 value, the 5th byte is actually 1E.
\
\
Now attacker can revert back to their original value of C1 and calculate original 5th byte of P2:
\
EE ⊕ 1E = F0
\
\
If you see the original value of P2 at 5th byte, it’s actually F0. So, attacker can recover original plaintext byte!
\
\
Attacker can continue, and make the padding equal to 5, and recover 4th byte of plaintext and so on…he can recover all the plaintext byte by byte using this method!
