# plainAES
The programm is designed to encrypt text or files with plain AES as a programming exersise.  
AES in CTR-Mode is used to extend AES to arbitary messages length.  
The final output is the contatination of some magic bytes as a header, the nonce and ciphertext.  
Keys are 256bit can be loaded from a file, passed in via the parogramm arguments or dirived from a password.  
The behaviour and command line flags are similar to openSSL and the command for openSSL for avcheving smilar resultas are documented below.

# Security
Using plain AES commes with some dangers.  
In CTR-Mode never reuse the nonce with the same key, the masasge is not authentificated.
More secure modes and more ussable are EAX and GCM which internally use CTR.
The Disadvantages:
* The Message length is leaked, often this is not considered confidantial
* Nonce, Key pair resuse, this is in pratise and danger, because of acesses of poor quality random generators. An the Crib Dragging Attack 
 on the two messages
* The message is not authenticated, e.g. the message can be modified with out detection. Alternevly AES-GCM hat an authentification mechanisam
* For extrem long messages the Pseudo random Function constructed from AES, in the CTR Mode chould be distinguised from a real random one.
 l ~ 2^(n/2) for l-Blocks with AES-n (z.B. AES-128 in der Exabyte range)
* For short messages the message size get inflatetd by the added nonce 8-12 Bytes and the header 6 Bytes.


The keys handling and lifecyle are very simple and dangerous.
The key handling is left to the user and keys have no limitation on the experiation, both are dangerous idears.
The keys need to be securly stored, the keyfile is not encrypted, maunelly entering the key in the command line leaks the key in the shell.
Leaking the key leaks all past and future mesasages.  

The key should be stored and entered in hexadecimal encoding, to have a more relatable represantaion of the key.  

A key can be derived from a passphrase/password. plainAES uses a simple key derivation via PBKDF2 with a fixed number of iterationans (10e5) and a fixed salt. Better key derivation algorithms exists and can be used.
# Usecases
Entering text and receving a copy-pastable output
`plainaes [-e or -d] -a -textin -pout -pass stdin`
Using AES for encrypting decrypting files
`plainaes [-e or -d] -in path_in -out path_out -k file:path_key`
Using plainaes as part of a bash pipeline
# Metadata

Metadata in the form of attributs like, timestamp, last edit, file name and EXIF data are not handled and lost on encryption.
Metadata that is part of a filecontent is encrypted and can be restored.


# Instalation

pycryptodome needs to be installed. On some distribution pyrcypto is installed and need to be removed because both use the same install name "Crypto".

pycrypto is not maintained since 2014  
pycryptodome names componats diffrently compared to pycrypto.

```
pip uninstall pycrypto
pip install pycryptodome
```

# Header

A Header is added in to the file, it includes the mode name and the IV or Nonce.
The header has free bits and is not in messagae authetification part of some AES Modes.
The header is composed as   MagicBytes || Nonce || ciphertext  (as concatenation).
The magic bytes are 'AESCTR' in binary after the base64 encoding i.e. hex 0044824D1 extended to 4 bytes 0044824D10. The magic bytes can indicate with AES-Mode and encoding is used, for now only CTR-mode is supported.
The nonce is 8 bytes long. The standard NIST SP 800-38A proposes 12 bytes for the nonce.
The nonce is not padded to 16 bytes, this can be an idea to increase comtibility between software. Nonce si generated by Pycryptodome and it random source.
The Nonce is not removed with the -noheader option.  

# Arguments
The Argument behavour is similar to openssl, notable plainAES does not expect an input nor generated an ouput via stdin/stdout by default, for this use the new flags -textin and -pout

-in filename
    This specifies the input file.

-out filename
    This specifies the output file. It will be created or overwritten if it already exists.

-e or -d
    This specifies whether to encrypt (-e) or to decrypt (-d). Encryption is the default. Of course you have to get all the other options right in order for it to function properly.

-a, -base64
    These flags tell plainAES to apply Base64-encoding before or after the cryptographic operation. The -a and -base64 are equivalent. If you want to decode a base64 file it is necessary to use the -d option.
The linebreak behaviour of base64 encoding comapred to openSSL base64 is diffrent.

-pass arg
    This specifies the password source. Possible values for arg are pass:password, file:filename or stdin, where password is your password and filename file containing the password. TODO test wether stdin is used by getpassword.

-key arg
    This option allows you to set the key or key source used for encryption or decryption in hexdecimal encoding. This is the key directly used by the cipher algorithm. If no key is given plainAES will derive it from a password.
    Possible values for arg are key:keyword or file:filename, where keyword is your key and filename file containing the key in the first line of the file in hexdecimal encoding.

-noheader
    this allows to remove the magic bytes. Then the output is only the nonce and ciphertext output form cipher

-pout
    This propmt plainAES to output the result in the commandline

-textin
    This enables the input of text via commandline
    
 -prompt
    This activates text prompt to explain input and output


# AES Modes

AES is a blockcipher for encrypting and decrypting block of size 128bits. Diffrent modes of AES are proposed for Streamcipher where messages can have arbitary length.
The following AES-modes use only the encryption routine Enc_K to generate psydorandom stream and XOR ing with the plaintext. The decryption routine then also uses only Enc_K from AES to reproduce the psydorandom stream and retrevive the plantext.

CTR-Mode  
Encrypt the successiv numbers ctr, ctr+1, ... and use this stream  

ctr_0 := Nonce + 00000000  
and incrimenting  
C_i = P_i XOR Enc_K(ctr_i)  


Output Feedback Mode (OFB)  
Encrypting the IV i times and uses this Stream  

C_i = P_i XOR Enc_K^i(IV)  


Cipher Feedback Mode (CFB)  
use the cipher text as the new IV  

C_0 = P_0 XOR Enc_K(IV)  
C_i = P_i XOR Enc_K(C_i-1)  

# Alternatives

openssl provides a similar functionaliy and by default has pager behavoir, expecting input from stdin and useing stdout for output.
Additional ciphers and encoder are supported and can use filediscriptors directly.
Example:
Encryption: `openssl aes-256-cbc -in attack-plan.txt -out message.enc`
Decryption: `openssl aes-256-cbc -d -in message.enc -out plain-text.txt`

For editing encrypted text files and editor can read from a named pipe and write to it.
The content will be decrypted and encrypted again. Named pipe are in Memory or Kernel memory and are not files, and dont leak the content to disk. Beware of the right who can open and read from the pipe.

# Test
unittest test the functionality of all function and include functional test.
The underlying file system is used to generate some files and read them for tesing purpuses. 

The test can be performed with

```
python -m unittest
```

TODO:
password verifcation in prompt
skript for named pipes and editor
keyring for managing keys






