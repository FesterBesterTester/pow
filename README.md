# pow - A Simple Interactive Password Manager



## Usage

This is an interactive command line tool written in python.

    Usage:
      pow.py [-f password_file]

    Interactive Commands:
      d: delete user
      g: get password for a site/user
      s: set password for a site/user
      m: set master password
      la: list all
      ls: list sites
      h: help
      q: quit

## Dependencies
 * pycrypto 2.6.1
 * py-bcrypt 0.4

## Design Decisions

### Encryption Library

With ~1000000 monthly downloads, the PyCrypto package seems to be the most widely adopted python cryptographic library. It provides random number generation suitable for cryptographic use, secure hash functions, and encryption algorithms.

### Cipher

AES was chosen due to its wide adoption and thorough peer review. There are several different encryption modes within AES. Choice of encryption mode was informed by NIST's [Recommendation for Block Cipher Modes of Operation](http://csrc.nist.gov/publications/nistpubs/800-38a/sp800-38a.pdf).

#### Overview of AES Encryption Modes

##### Electronic Code Book (ECB)

ECB does not use an initialization vector (IV), and each block is encrypted independently. This makes ECB highly parallelizable and facilitates quick access and updates to individual blocks. Another implication is that with a given key, a given plaintext block always gets encrypted to the same ciphertext block, which can be undesirable in some circumstances. PyCrypto says this mode is not recommended due to possible exposure of plaintext symbol frequency. The length of the plaintext must be a multiple of the block size.

##### Cipher Block Chaining (CBC)

CBC does use an IV. During encryption, each plaintext block is combined with the ciphertext of the previous block, making encryption of a block dependent on all previous blocks. Due to these inter-block dependencies, multiple blocks cannot be encrypted in parallel. To update an encrypted sequence of blocks when a plaintext block is changed, all blocks after the updated block must be re-encrypted. Decryption can be done in parallel since the cipher text of only the block to be decrypted and the previous block are required as inputs. The length of the plaintext must be a multiple of the block size.

##### Cipher Feedback Mode (CFB)

CFB does use an IV. The length of the plaintext must be a multiple of the segment size. CFB segment size is defined in bits. CFB is similar to CBC in that to encrypt a segment, the ciphertext of the previous segment is combined with the current plaintext segment. This makes it impossible to encrypt multiple blocks in parallel. Decryption depends on the ciphertext of the current and previous segments, making decryption parallelizable.

##### Output Feedback Mode (OFB)

OFB requires an IV that may only be used once for a given key. The length of the plaintext is not required to be a multiple of the block size.

$#### Counter Mode (CTR)

To do: add CTR description here.

## Implementation

The password database is encrypted with AES (mode CFB). The AES key is a bcrypt hash of the master password. Using a bcrypt hash rather than the master password directly provides two advantages. It gives us the 32 bytes needed for an AES256 key even if our password is not 32 bytes, and it greatly slows down brute force attacks due to the inherently slow nature of bcrypt hashing. Since the bcrypt hash is only 31 bytes and we need 32 bytes for the key, the final byte of bcrypt salt is also used in the key.

To decrypt the file, the user is asked for the master password. The provided password is then hashed using the same salt used during encryption. If the provided password is the same as the password used for encryption, the resulting bcrypt hash will also be the same, allowing the file to be decrypted.

The file format is as follows:

      +-----------------------------------------------------------+
      |  $<bcrypt alg/format>$<bcrypt cost param>$<bcrypt salt>   |
      |    1 or 2 characters  |  2 characters    | 22 characters  |
      +---+---------------------------------------------------+---+
      | A |                  SHA256 of payload                | A |
      | E +---------------------------------------------------+ E |
      | S |                                                   | S |
      |   |                                                   |   |
      | E |                                                   | E |
      | N |                                                   | N |
      | C |                                                   | C |
      | R |                      payload                      | R |
      | Y |                                                   | Y |
      | P |                                                   | P |
      | T |                                                   | T |
      | E |                                                   | E |
      | D |                                                   | D |
      +---+---------------------------------------------------+---+

The first 28 or 29 bytes contain the bcrypt algorithm, format, and salt used with the master password. The format is the same as that returned from bcrypt.gensalt().

* $2$, $2a$ or $2y$ identifying the hashing algorithm and format
* a two digit value denoting the cost parameter, followed by $
* 22 characters of salt (effectively only 128 bits of the 132
              decoded bits)

The remainder of the file is AES encrypted. Once decrypted, the encrypted portion begins with the SHA256 of the payload followed by the payload itself. The correctness of the password supplied for decryption can be determined by comparing the included SHA256 with the SHA256 of the decrypted payload. If they match, the password is correct.

The payload format is json as follows:

    {
      "site1": {
        "user1": {
          "pw": "password",
          "note": "note"
        },
        "user2": {
          "pw": "password",
          "note": "note"
        }
      },
      "site2": {
        "user1": {
          "pw": "password",
          "note": "note"
        },
        "user2": {
          "pw": "password",
          "note": "note"
        }
      }
    }

