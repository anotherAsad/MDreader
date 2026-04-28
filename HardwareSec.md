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
        printf("=============== CORRUPTIBLE_LOC: %d ===============\n", CORRUPTIBLE_LOC);

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
   - Keep candidate set from first fault, intersect with candidate sets from subsequent faults using `to_keep[]` array, and stop when only one candidate remains.
   - Store unique result in `M9_recovered[CORRUPTIBLE_LOC]` and set `SIG_M9_RECOVERED = 1`.

   In code, this looks like:
   ```C
   if(HIJACK) {
      printf("\n\t\t:: [Searching for cyphertext anomaly...]\n");
      int non_zero_idx  = -1;
      uint8_t non_zero_byte = 0;
      state_t new_state = {0};

      for(int i=0; i<16; i++) {
         new_state[i/4][i%4] = correct_cyphertext[i] ^ buf[i];

         if(non_zero_idx == -1 && new_state[i/4][i%4] != 0) {
            non_zero_idx = i;
            non_zero_byte = new_state[i/4][i%4];
         }
      }

      print_state(new_state, " [Diff State]        : ");

      InvShiftRows(&new_state);

      print_state(new_state, " [Post InvShiftRows] : ");

      // try the byte values:
      printf("\t\t:: non_zero_idx: %d, non_zero_byte: 0x%02X\n", non_zero_idx, non_zero_byte);
      printf("\t\t:: Byte values that satisfy `SB(M9) ^ SB(M9 ^ e) = %02X`\n\n", non_zero_byte);

      static int plausible_M9[14];
      static int plausible_M9_count;
      
      int to_keep[14] = {0};
      int count = 0;

      // reset the plausible bytes list
      if(CORRUPTIBLE_BYTE == 0x01) {
         plausible_M9_count = 0;

         for(int i=0; i<14; i++)
            plausible_M9[i] = 0;
      }

      // Iterate over all possible byte values and see if the equation holds.
      // If it does, list all possible candidates. Keep discarding candidates
      // from other single bit-faults till we are only left with one candidate.
      // That is the true M9 byte.
      for(unsigned byte = 0; byte < 256; byte++) {
         if(non_zero_byte == (getSBoxValue(byte) ^  getSBoxValue(byte ^ CORRUPTIBLE_BYTE))) {
            if(byte % 8 == 0)
               printf("\n");
            
            printf("\t\t:: %02X", byte);

            if(CORRUPTIBLE_BYTE == 0x1) {
               plausible_M9[plausible_M9_count++] = byte;
            }
            else {
               for(int i = 0; i < plausible_M9_count; i++) {
                  if(byte == plausible_M9[i])
                     to_keep[count++] = byte;
               }
            }
         }
      }

      if(CORRUPTIBLE_BYTE != 0x01) {
         // copy back
         for(int i = 0; i < count; i++) {
            plausible_M9[i] = to_keep[i]; 
         }
         
         plausible_M9_count = count;
      }

      // List the plausible M9. If it is only one, 
      if(plausible_M9_count == 1) {
         printf("\n\t\t:: [Single M9 Byte found] : %02X\n", plausible_M9[0]);
         M9_recovered[CORRUPTIBLE_LOC] = plausible_M9[0];
         SIG_M9_RECOVERED = 1;
      }

      printf("\n");
   }
   ```

4) Added round-key reconstruction helper:
   - Here, we use the known $M_9$ to get the last round key. The equation we use is:
   $$$K_{10} = C \oplus ShiftRows(SubBytes(M_9))$$$
   - `get_key_from_M9()` computes:
     ```Python
     K10 = C ^ ShiftRows(SubBytes(M9))
     ```
   - Prints recovered K10 bytes.

   Looks like:
   ```C
   void get_key_from_M9() {
      //    K_10 = C xor SR(SB(M_9))
      state_t M9_subbed_rowshifted;

      uint8_t key_10[16];

      for(int i=0; i<16; i++)
         M9_subbed_rowshifted[i/4][i%4] = M9_recovered[i];

      SubBytes(&M9_subbed_rowshifted);
      ShiftRows(&M9_subbed_rowshifted);

      for(int i=0; i<16; i++)
         key_10[i] = correct_cyphertext[i] ^ M9_subbed_rowshifted[i/4][i%4];
      
      printf("[KEY_10]        : ");

      for(int i=0; i<16; i++)
         printf("%02X ", key_10[i]);

      printf("\n");
   }
   ```



How this matches assignment steps
---------------------------------
- Step 1: Single-bit fault injected in M9 before last round.
- Step 2: Original/faulty ciphertexts compared.
- Step 3-5: Byte-wise M9 guessing via S-box relation and intersection across multiple faults.
- Step 6: Repeat for all 16 bytes.
- Step 7: Recover K10 from recovered M9 and correct ciphertext.

Run output file
---------------
- Results captured in `res_hijack`. They look like:
```
ECB encrypt: ============================================ CORRUPTIBLE_LOC: 15 ============================================

		:: [Searching for cyphertext anomaly...]
		:: [Diff State]        : 000000ba000000000000000000000000
		:: [Post InvShiftRows] : 000000000000000000000000000000ba
		:: non_zero_idx: 3, non_zero_byte: 0xBA
		:: Byte values that satisfy `SB(M9) ^ SB(M9 ^ e) = BA`

		:: C4		:: C5
ECB encrypt: 
		:: [Searching for cyphertext anomaly...]
		:: [Diff State]        : 000000a8000000000000000000000000
		:: [Post InvShiftRows] : 000000000000000000000000000000a8
		:: non_zero_idx: 3, non_zero_byte: 0xA8
		:: Byte values that satisfy `SB(M9) ^ SB(M9 ^ e) = A8`

		:: C4		:: C6
		:: [Single M9 Byte found] : C4

ECB encrypt: ============================================ FINAL RESULT ============================================
		Recovered M9: BB 36 C7 EB 88 33 4D 49 A4 E7 11 2E 74 F1 82 C4 
 :: [KEY_10]  : D0 14 F9 A8 C9 EE 25 89 E1 3F 0C C8 B6 63 0C A6 
```
