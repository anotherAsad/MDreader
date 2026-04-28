<h1>Assignment 5 - DFA Fault Attack on tiny-AES-c</h1>
<h2>Change Log</h2>

Overview of Changes
-------------------
This implementation adds a HIJACK mode to tiny-AES-c to emulate a __Differential Fault Analysis (DFA)__ attack on AES-128 by injecting single-bit faults into $M_9$ (the state before the final AES round), recovering $M_9$ byte-by-byte, and reconstructing the round-10 key $K_{10}$.

Files Changed
-------------
- `test.c`
- `aes.c`

What I changed in test.c
------------------------
1) Added global controls/state for fault injection and recovery:
   - `HIJACK` flag to toggle normal AES test mode vs DFA mode.
   - `correct_cyphertext[]` storing the known correct ciphertext for the fixed AES-128 test vector.
   - `CORRUPTIBLE_BYTE` and `CORRUPTIBLE_LOC` to select injected fault value/location.
   - `M9_recovered[16]` to store recovered bytes of $M_9$.
   - `SIG_M9_RECOVERED` as a per-byte convergence signal.
  
  In the code, these lines appear as follows:
   ```C
    int HIJACK = 0;
    uint8_t correct_cyphertext[] = {
      0x3a, 0xd7, 0x7b, 0xb4, 0x0d, 0x7a, 0x36, 0x60,
      0xa8, 0x9e, 0xca, 0xf3, 0x24, 0x66, 0xef, 0x97
    };
    uint8_t CORRUPTIBLE_BYTE = 0x0;
    uint8_t CORRUPTIBLE_LOC = 0x1;
    uint8_t M9_recovered[16] = {0};
    char SIG_M9_RECOVERED = 0;
   ```

2) Modified program flow in `main()`:
   - If `HIJACK == 0`: run standard ECB test behavior.
   - If `HIJACK == 1`:
     - Iterate `CORRUPTIBLE_LOC` from 0 to 15 (recover one $M_9$ byte at a time).
     - For each location, inject bit faults using `CORRUPTIBLE_BYTE`. The fault locations are `{0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80}`.
     - Run AES encryption and stop early for any byte if a unique candidate is found.
     - After all 16 bytes are recovered, print full recovered $M_9$ and derive __K10__ using `get_key_from_M9()`.

  In the code, these lines appear as follows:
  ```C
      if(!HIJACK) {
        test_encrypt_ecb();
        //test_encrypt_ecb_verbose();
        return exit;
    }


    // Iterate over all 16 bytes (CORRUPTIBLE_LOC) and 8 bits within each byte to introduce errors.
    for(CORRUPTIBLE_LOC = 0; CORRUPTIBLE_LOC < 16; CORRUPTIBLE_LOC++) {
        printf("=============== CORRUPTIBLE_LOC: %d ("===============\n", CORRUPTIBLE_LOC);

        SIG_M9_RECOVERED = 0;

        for(CORRUPTIBLE_BYTE = 0x1; CORRUPTIBLE_BYTE !=  0x0; CORRUPTIBLE_BYTE <<= 0x1) {
            test_encrypt_ecb();

            if(SIG_M9_RECOVERED)
                break;
        }
    }

    printf("=============== FINAL RESULT ================\n");
    printf("[Recovered M9]  : ");
    
    for(int i=0; i<16; i++)
        printf("%02X ", M9_recovered[i]);

    printf("\n");

    get_key_from_M9();

    return exit;
  ```

3) Modified test_encrypt_ecb():
   - In `HIJACK` mode, bypasses pass/fail comparison and returns 0 so that failure log due to DFA is suppressed. This change is not really necessary.

What I changed in aes.c
-----------------------
1) Imported externally defined HIJACK/DFA globals from test.c:
   - `HIJACK`, `correct_cyphertext`, `CORRUPTIBLE_BYTE`, `CORRUPTIBLE_LOC`, `M9_recovered`, `SIG_M9_RECOVERED`.
  
  This is done using `extern` keyword:
  ```C
  extern int HIJACK;
  extern uint8_t correct_cyphertext[];

  extern uint8_t CORRUPTIBLE_BYTE;
  extern uint8_t CORRUPTIBLE_LOC;
  extern uint8_t M9_recovered[];
  extern char SIG_M9_RECOVERED;
  ```

2) Injected fault at the beginning of the final round (on M9):
   - In `Cipher()`, after round Nr-1 AddRoundKey and before final-round SubBytes/ShiftRows+AddRoundKey, flips one chosen bit:
     ```state[CORRUPTIBLE_LOC/4][CORRUPTIBLE_LOC%4] ^= CORRUPTIBLE_BYTE;```
   - This exactly emulates a 1-bit fault model used by theoretical DFA.

  This is done as follows in the code:
  ```C
  if(round == Nr - 1) {
    if(HIJACK) {
      // print_state(*state, " [Original M9] : ");        
      (*state)[CORRUPTIBLE_LOC/4][CORRUPTIBLE_LOC%4] ^= CORRUPTIBLE_BYTE;
    }
  }
  ```

3) Added post-encryption DFA analysis in `AES_ECB_encrypt()` (HIJACK mode):
   - Compute ciphertext difference state:
     __Diff = correct_cyphertext XOR faulty_ciphertext.__
   - Apply InvShiftRows(Diff) to map the non-zero difference back to the corresponding $M_9$ byte position.
   - For all guesses $x$ in $[0..255]$, test DFA equation:
     $SBox(x) \oplus SBox(x \oplus e) == NonZeroByte$
     where __e__ = CORRUPTIBLE_BYTE.
   - If any byte $x$ satisfies the equation, it is a possible byte of $M_9$.
   - Keep candidate set from first fault, intersect with candidate sets from subsequent faults, and stop when one candidate remains.
   - Store unique result in M9_recovered[CORRUPTIBLE_LOC] and set SIG_M9_RECOVERED = 1.

4) Added round-key reconstruction helper:
   - get_key_from_M9() computes:
     K10 = C XOR ShiftRows(SubBytes(M9))
   - Prints recovered K10 bytes.

How this matches assignment steps
---------------------------------
- Step 1: Single-bit fault injected in M9 before last round.
- Step 2: Original/faulty ciphertexts compared.
- Step 3-5: Byte-wise M9 guessing via S-box relation and intersection across multiple faults.
- Step 6: Repeat for all 16 bytes.
- Step 7: Recover K10 from recovered M9 and correct ciphertext.

Run output file
---------------
- Results captured in `res_hijack`.
